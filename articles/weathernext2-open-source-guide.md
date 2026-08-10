---
title: "WeatherNext 2公開コードを試す前に知るべき実行コスト"
emoji: "🌦️"
type: "tech"
topics: ["ai", "python", "google", "weather", "github"]
published: true
---

2026年8月11日のGitHub Trendingでは、Google DeepMindの[WeatherNext](https://github.com/google-deepmind/weathernext)が確認時点で327 stars todayを集めていました。8月7日にはv0.3.0が公開され、WeatherNext 2（WN2）のモデルコード、設定、Colab notebookが追加されています。

結論から言うと、WeatherNext 2を試す方法は2つに分けるべきです。

1. **予報データを使いたい**：BigQuery、Earth Engine、Google Cloud Storageの公開フィードを読む
2. **モデル自体を研究したい**：v0.3.0を固定し、ColabのTPUまたは十分なVRAMを持つGPUで推論する

天気データをアプリで利用したいだけなら、最初から重みをダウンロードしてモデルを動かす必要はありません。この記事では、公式README、開発者ドキュメント、v0.3.0のソースを基に、公開範囲、必要な計算資源、再現手順、運用上の注意を整理します。

:::message
GitHub Trendingとstar数は時間によって変動します。本記事の数値は2026年8月11日7時1分（JST）のスナップショットです。Trending入りは注目度の参考であり、予報精度、品質、安全性、本番適合性の保証ではありません。
:::

## 確認した公開状況

GitHub API、Release、固定tagのソースから次を確認しました。

| 項目 | 確認結果 |
| --- | --- |
| リポジトリ | `google-deepmind/weathernext` |
| 累計stars | 7,323 |
| 当日増加 | 327 stars today |
| 最新安定Release | v0.3.0 |
| v0.3.0公開日時 | 2026年8月7日0時1分（JST） |
| v0.3.0 commit | `89c4b2a77a1c57b328b909c575550fd2e5aadc9c` |
| v0.3.0の変更 | WN2 supportを追加 |
| ライセンス | コードとnotebookはApache-2.0、その他資料はCC BY 4.0 |
| package分類 | Development Status :: 3 - Alpha |
| 公式サポート | 公式Google製品ではない研究コード |

main branchは調査時点で`0.3.1.dev`へ進んでいました。再現性が必要なら、mainではなくv0.3.0または40桁のcommit SHAを固定します。

## WeatherNext 2が提供するもの

WeatherNextは、Google DeepMindとGoogle Researchが開発する全球・中期の大気予報モデル群です。現在のリポジトリは、次をまとめたホームになっています。

- WeatherNext 2：Functional Generative Network（FGN）を使う現行モデル
- WeatherNext Cyclones：熱帯低気圧予報向けcheckpointとtracker
- WeatherNext Graph：旧GraphCast
- WeatherNext Gen：旧GenCast

公式モデル仕様では、WeatherNext 2は次の構成です。

| 項目 | WeatherNext 2 |
| --- | --- |
| 空間解像度 | 0.25度、赤道付近で約30km |
| 時間解像度 | 公開フィードは6時間 |
| 初期化 | 00、06、12、18 UTC |
| 予報期間 | 15日 |
| 予報形式 | 64 memberのensembleとensemble mean |
| 対象 | 全球 |
| 主な変数 | 気温、風、湿度、ジオポテンシャル、鉛直速度、気圧、降水など |

公式ドキュメントは、WeatherNext 2が旧WeatherNext Genに対し、変数と0〜15日のlead timeの99.9%で上回ると説明しています。ただし、これはGoogleが公開したモデル評価の集計です。自分の用途では、地点、季節、現象、lead time、損失関数を固定して再評価する必要があります。

またGoogleの2025年11月の発表には「最大1時間解像度」とありますが、開発者ドキュメントでは公開フィードの時間解像度は6時間で、1時間timestepはVertex AIのexperimental capabilityとして区別されています。記事や設計書では、**モデルが持つ能力、公開データの粒度、利用中のproduct surface**を混同しないことが重要です。

## v0.3.0で公開されたモデルの違い

READMEは、WeatherNext 2とWeatherNext Cyclonesの複数checkpointを案内しています。

### WeatherNext 2

`WeatherNext2_<2025_model{1,2,3,4}.npz`は0.25度版で、2024年までのデータで学習され、運用HRES初期条件から使う設計です。WeatherNext Cyclonesとの主な出力差として、100m風も予測します。

### WeatherNext Cyclones

2025、2024、2023を評価対象とするcheckpoint群があります。`WeatherNextCyclones_<2025`は、2025年のAtlantic hurricane seasonでlive運用されたモデルに対応します。

### WeatherNext Cyclones Mini

Miniは1度解像度で、より小さい計算資源向けです。ただしREADMEは、大きなモデルと同等の性能を期待すべきではないと明記しています。

「Miniで動いた」ことは、0.25度版の性能や計算量を確認したことにはなりません。最初の目的をAPI利用、コード理解、Mini推論、論文再現のどれかに分けます。

## まず公開予報フィードを使う

公式ドキュメントは、運用版WeatherNext 2のreal-timeおよびhistorical forecastを3つのGoogle surfaceで提供しています。

| Surface | 向いている用途 | データ |
| --- | --- | --- |
| BigQuery | SQL集計、業務データとの結合、大規模分析 | WeatherNext 2、ensemble mean |
| Earth Engine | 地理空間解析、map表示、raster処理 | WeatherNext 2、ensemble mean |
| Google Cloud Storage | xarray、機械学習、独自batch処理 | Zarr形式 |

GCS上の公式pathは次です。

```text
gs://weathernext/weathernext_2_0_0/zarr
gs://weathernext/weathernext_2_0_0_mean/zarr
```

アプリで平均予報だけが必要なら、64 memberをすべて取得する前にensemble meanを検討します。不確実性やworst-case scenarioを扱うなら、meanだけでは分布情報が失われるため、memberごとのデータが必要です。

実務では次の順序が安全です。

1. 公式starter notebookでschemaと単位を確認する
2. 1つの初期化時刻、1変数、狭い緯度経度へ絞る
3. 欠損、座標系、経度表現、時刻、単位を検証する
4. 取得量とquery costを確認する
5. 既存の観測・公式予報と用途別に比較する
6. 必要になった場合だけmember数と対象領域を増やす

大量データを最初から全期間downloadすると、計算より転送・保存が先に問題になります。

## モデルを自分で動かす場合の現実的な条件

READMEはTPUでの実行を推奨しています。実装がTPU向けに最適化されているためです。

GPUを使う場合はattention implementationの切替が必要です。必要資源の目安として、公式READMEは次を挙げています。

- non-Mini：VRAMのためH100が必要
- Mini：P100で推論可能な見込み
- Colabの既定demo：WeatherNext Cyclones Miniと`v5e-1` runtime
- 他の大規模model：`v5p` acceleratorが必要

ここでの表現は保証ではありません。入力期間、batch、member数、JAX/XLA版、driver、compile cacheで使用量は変わります。

full trainingにはERA5が必要で、運用fine-tuningにはWeatherBench2のHRES dataが案内されています。モデルコードがApache-2.0でも、学習データ、重み、第三者データには別の条件が適用される可能性があります。

## 固定tagを読み取り専用で検査する

推論環境を作る前に、ソースだけを検査できます。

```bash
workdir="$(mktemp -d)"
git clone --depth 1 --branch v0.3.0 \
  https://github.com/google-deepmind/weathernext.git \
  "$workdir/weathernext"

cd "$workdir/weathernext"
git rev-parse HEAD
python3 -m compileall -q weathernext
```

期待するcommitは次です。

```text
89c4b2a77a1c57b328b909c575550fd2e5aadc9c
```

記事執筆時にこのtagをcloneし、`weathernext/`配下のPython 70ファイルへ`compileall`を実行したところ、構文検査は成功しました。`*_test.py`は6ファイルありました。

:::message
`compileall`はPython構文を確認するだけです。JAX、TPU/GPU、model weights、sample dataを使ったimport、unit test、推論結果は確認していません。依存導入や実行成功を示すものではありません。
:::

`setup.py`ではv0.3.0に次の制約があります。

- Python 3.10 / 3.11をclassifierに記載
- package statusはAlpha
- `xarray<=2026.2.0`
- `xarray_jax`はGitHubのv0.1.1を参照
- JAX、Haiku、Dask、xarray、dinosaur-dycoreなど多数の依存

依存を入れるときは、通常環境へ直接入れず、使い捨てのvirtual environmentまたはColab runtimeを使います。

## 公式手順で版を固定する

READMEのインストール例は次です。

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install \
  "git+https://github.com/google-deepmind/weathernext.git@v0.3.0"
```

ただし、これだけでは推論できません。別途model weightsと初期状態データが必要です。GPUではnotebookに従ってattention implementationを変更します。

最も短い公式経路は`docs/weathernext2/wn2_demo.ipynb`です。notebookでは次の流れを確認できます。

1. Google Cloud Storageからmodel weightsを取得
2. HRESなどの初期状態を読む
3. FGN architectureを初期化
4. autoregressive rolloutで予測
5. 気温、風、ジオポテンシャルなどを可視化
6. cyclone trackerを実行
7. loss計算とgradient stepを確認

自分の結果を報告するときは、少なくともtag、weight filename、入力時刻、初期条件、accelerator、JAX版、member数、lead timeを残します。

## 予報を検証する単位

WeatherNext 2はprobabilistic forecastです。1つの地点の1回の予報が当たったかだけでは、ensembleの価値を評価できません。

用途別に検証単位を変えます。

| 用途 | 確認したい指標・条件 |
| --- | --- |
| 日常の気温 | MAE、bias、地点、lead time、季節 |
| 降水 | threshold別precision / recall、地域、時間窓 |
| 強風 | exceedance probability、最大風速、観測網 |
| cyclone | track error、強度誤差、basin、lead time |
| 発電予測 | ensemble spread、calibration、設備位置 |
| 物流 | 遅延costに合わせたdecision metric |

Googleが示す99.9%の比較は重要な全体指標ですが、自分の意思決定で必要な閾値を直接保証しません。予報モデルのスコアと、最終的な業務KPIを分けます。

## 公式警報の代わりに使わない

READMEには明確なdisclaimerがあります。

- WeatherNextは実験的なresearch project
- 公式にサポートされたGoogle productではない
- API stabilityの保証はなく、breaking changeがあり得る
- 政府の気象機関と共同制作・endorseされたものではない
- 公式のalert、warning、noticeを置き換えない

したがって、災害対応や人命に関わる表示では、WeatherNext出力だけで避難・運休・操業停止を決定しません。気象庁など管轄機関の公式警報を正として、AI予報は補助情報として出典、初期化時刻、lead time、不確実性を表示します。

## 本番利用前のチェックリスト

- [ ] v0.3.0またはcommit SHAを固定した
- [ ] 予報フィード利用と自前推論のどちらが目的か決めた
- [ ] ensembleとensemble meanを用途に応じて選んだ
- [ ] 初期化時刻、lead time、単位、座標系を確認した
- [ ] データ転送量、保存量、query costを見積もった
- [ ] acceleratorとVRAMの条件を確認した
- [ ] model weightsと第三者dataの利用条件を確認した
- [ ] 自分の地域・季節・現象でbacktestした
- [ ] 欠損・遅延・配信停止時のfallbackを用意した
- [ ] 公式警報を置き換えないUIと運用にした
- [ ] source version、入力、出力、評価結果を記録した

## まとめ

WeatherNext v0.3.0は、WeatherNext 2とCyclonesの研究コード、設定、notebookを公開し、モデルの中身を調べたり固定版で実験したりできるようにしました。

一方、実務で予報データを使うだけなら、まずBigQuery、Earth Engine、GCSの公開フィードを選ぶ方が合理的です。モデルを動かす経路は、Miniでもaccelerator、weights、初期条件、JAX環境が必要で、non-MiniはH100またはTPU級の資源を前提とします。

採用判断では次を分けてください。

1. Trendingの注目度
2. 公開コードの再現性
3. Googleが示すモデル全体の評価
4. 自分の地域・現象での精度
5. 実運用の配信・費用・警報責任

AI気象モデルの価値は大きい一方、良いスコアや公開コードだけで業務判断に直結させるべきではありません。最小の公開データqueryから始め、自分の用途で検証してから対象領域と計算量を広げるのが安全です。

## 参考リンク

- [WeatherNext公式リポジトリ](https://github.com/google-deepmind/weathernext)
- [WeatherNext v0.3.0 Release](https://github.com/google-deepmind/weathernext/releases/tag/v0.3.0)
- [WeatherNext models（Google Developers）](https://developers.google.com/weathernext/guides/models)
- [Accessing WeatherNext forecasts](https://developers.google.com/weathernext/guides/access-forecast)
- [WeatherNext 2公式ブログ](https://blog.google/innovation-and-ai/models-and-research/google-deepmind/weathernext-2/)
- [WeatherNext Cyclones論文](https://www.nature.com/articles/s41586-026-10953-2)
- [FGN technical report](https://arxiv.org/abs/2506.10772)
- [GitHub Trending](https://github.com/trending?since=daily)
