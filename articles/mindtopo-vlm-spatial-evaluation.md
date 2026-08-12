---
title: "MindTopoから学ぶVLM空間推論評価：認識と計画を分ける"
emoji: "🪢"
type: "tech"
topics: ["ai", "vlm", "benchmark", "research", "github"]
published: false
---

2026年8月12日、Microsoft Researchは、Vision-Language Model（VLM）のトポロジー理解を測るベンチマーク「MindTopo」を紹介しました。

結論から言うと、MindTopoが示す重要な点は、**1枚の画像で構造を認識できることと、その構造を壊さず複数手順を計画できることは別の能力**だということです。評価対象モデルの最高Overallは54.13%で、人間の97.40%を大きく下回りました。モデル別に見ると、静的なReasoningより対話的なPlanningが弱い傾向も確認できます。

この記事では、公式プロジェクトページ、Microsoft Researchの発表、公開データ、公式GitHubリポジトリを基に、次を整理します。

- 距離や方向とは違う「トポロジー」をどう評価するか
- 13タスクと11,016 instancesの構成
- Overallだけでは見えないReasoningとPlanningの差
- 公開コードを試す前に確認すべき再現性とライセンス
- ロボットやComputer Use Agentの評価へ応用する方法

:::message
MindTopoは2026年8月13日7時台（JST）に確認した研究ベンチマークです。評価値は特定のモデル版、task、prompt、実行条件に依存します。単一の順位を一般的なVLM性能や本番安全性の保証として使わないでください。
:::

## まず公開状況を区別する

確認できた一次情報は次のとおりです。

| 項目 | 確認結果 |
| --- | --- |
| Microsoft Research発表 | 2026年8月12日 |
| 研究主体 | Northwestern University、Microsoft Research、Stanford Universityの研究者 |
| 評価対象 | 5つのtopological property、13 task types |
| 規模 | 11,016 instances、11 MLLMs |
| 公式データ | Hugging Faceで公開、CC BY 4.0 |
| 公式コード | `mll-lab-nu/MindTopo`で公開 |
| コードのRelease | GitHub Releaseなし |
| コードのライセンス | GitHub上で明示的なlicense file・license metadataを確認できず |
| 論文 | プロジェクトページのcitationは`misc`で、arXiv IDの記載なし |
| コードの状態 | READMEは「minimal snapshot」と説明 |

データがCC BY 4.0だからといって、リポジトリ内のコードも同じ条件だとは限りません。コードを再配布、改変して製品へ組み込む場合は、権利者へ利用条件を確認する必要があります。

またGitHubリポジトリにはtagやReleaseがありません。公式READMEの手順を再現するときは、調査時点のmain commit `c159c95db95c425564758d6dcab8e68e4379084e`を記録してください。

## トポロジーは「正確な座標」ではなく「変形しても残る関係」

一般的な空間評価は、左右、上下、距離、角度、大きさ、相対位置を問います。MindTopoが扱うのは、物体を曲げたり伸ばしたりしても維持される構造的な関係です。

たとえば、ロープを曲げても結び目は同じです。部屋の形が変わっても、通路が閉じられなければ接続関係は残ります。柵の輪郭が歪んでも、羊が内側か外側かという関係は維持されます。

MindTopoはPiagetの認知研究に着想を得て、次の5分類を使います。

| 性質 | 問うこと | 例 |
| --- | --- | --- |
| Continuity | 経路や物体が途切れずつながっているか | maze、pipe |
| Separation | 1つの構造か、分離した複数の部分か | furniture assembly、one stroke |
| Order | 経路上の並び順が変形後も保たれるか | beads、origami、swap |
| Enclosure | 境界が内側と外側を作っているか | sheep and fence、hole、Chat Noir |
| Knots | 本当に結ばれているか、見た目だけ絡んでいるか | knot、untangle |

これは画像分類だけの問題ではありません。現実のagentは、接続・囲い・順序・結び目を認識した後、その関係を維持または変更する行動を選ぶ必要があります。

## 13タスクをReasoningとPlanningへ分ける

MindTopoは同じ5分類を2つの認知レベルで評価します。

### Reasoning：静的sceneを読む

8つのReasoning taskでは、1枚または複数のrendered sceneを見て構造を答えます。

- 2D Maze、3D Maze
- IKEA
- Bead、Origami Point
- Sheep、Hole
- Knots

失敗は、壁、開口、交差、境界などを画像から読み落とすperception errorとして現れやすくなります。

### Planning：複数actionで関係を操作する

5つのPlanning taskでは、simulatorと対話しながらaction sequenceを作ります。

- Pipe
- One Stroke
- Swap
- Chat Noir
- Untangle

環境は合法なactionを強制します。たとえば、ロープを別のロープの中へすり抜けさせるような物理的に許されない近道では解けません。

Planningでは、最初のsceneを理解していても途中で目的を失う、局所的には良さそうな手を選んで将来の接続を壊す、環境に存在しないactionを提案する、といった失敗を分離できます。

```text
静的sceneを認識
  ↓ Reasoningが測る範囲
現在のtopological stateを作る
  ↓
合法actionを選ぶ
  ↓
変化後のstateを更新する
  ↓
不変にすべき関係を維持する
  ↓ Planningが追加で測る範囲
目標stateへ到達
```

1枚のスクリーンショットへの正答率だけでは、この状態追跡を評価できません。

## データ構成を読む

公式プロジェクトページは11,016 instancesを、Reasoning 73%、Planning 27%と説明しています。8つのReasoning taskはそれぞれ約1,000 instances、5つのPlanning taskはそれぞれ600 instancesです。

すべてcontrolled simulatorから手続き的に生成され、難易度を調整できます。この設計には2つの利点があります。

1. simulatorの内部状態からexact ground truthを得られる
2. 見た目の複雑さと構造的な難しさを別々に変えられる

一方、手続き生成であることは現実世界への汎化を保証しません。renderer、object、camera、promptの規則を学習しただけではないか、別の生成条件や実画像でも確認する必要があります。

公開Hugging Face datasetには13 configsがあり、静的taskとplanning taskの`question.jsonl`、一部taskのzipが含まれます。dataset card上のtask categoryはvisual-question-answeringとimage-text-to-text、言語は英語です。

## OverallだけでなくReasoningとPlanningを読む

公式leaderboardは、Overallを13 task、Reasoningを8 task、Planningを5 taskの**unweighted mean**として計算しています。instance数で重み付けしたmicro averageではありません。

代表的な結果は次のとおりです。

| Model | Overall | Reasoning | Planning | R-P差 |
| --- | ---: | ---: | ---: | ---: |
| GPT-5.5 | 54.13 | 57.74 | 48.33 | 9.41 |
| Gemini 3.1 Pro | 53.54 | 60.00 | 43.20 | 16.80 |
| Gemini 3.1 Flash Lite | 36.31 | 50.84 | 13.06 | 37.78 |
| GPT-5.4 mini | 25.47 | 35.20 | 9.90 | 25.30 |
| Human | 97.40 | 95.77 | 100.00 | -4.23 |

R-P差は記事執筆時に表の値から`Reasoning - Planning`を再計算しました。4つの掲載モデルではPlanningがReasoningを下回りますが、差の大きさはモデルごとに異なります。

さらにtask単位では得意不得意が大きく違います。GPT-5.5はSwapで100.00%ですが、Pipeは21.67%、Untangleは33.30%です。Overall 54.13%という1つの値だけで、「空間推論が半分程度できる」と解釈してはいけません。

### 人間との差も一様ではない

Human rowはOverall 97.40%、Planning 100.00%です。一方、Reasoningは95.77%で、Knotsは91.55%です。benchmark自体にも人間が間違えやすいsceneがあります。

人間との差を報告するときは、model-humanのOverall gapだけでなく、task別の難しさ、回答者数、評価手順も確認すべきです。現在のproject pageだけではhuman evaluationの全条件を読み切れないため、公開予定の論文や追加資料が出たら更新が必要です。

## 画像・動画生成はworld modelの保証にならない

研究では、image generationとvideo rolloutがtopologyの維持に役立つかも調べています。Microsoft Researchの説明では、image generationは関係が1 frameで見える場合に助けになることがある一方、複数の交差やmoveをまたぐと不安定でした。video rolloutも、途中でtopologyを変えたりtask dynamicsへ違反したりしました。

ここから得られる教訓は、「未来画像を生成できる」ことと「合法な未来状態を予測できる」ことを分けることです。

```text
きれいなfuture frame
    ≠ 物理的に到達可能なstate
    ≠ 接続・順序・囲いを保存したstate
    ≠ goalへ近づくaction
```

Computer Use Agentで画面遷移を生成・予測させる場合も、見た目だけでなく、実行可能action、resource state、権限、不可逆な副作用を別のvalidatorで確認する必要があります。

## 公開コードを試す最小経路

公式READMEはPython 3.10以上3.13未満と`uv`を前提にしています。再現性のため、mainをそのまま追わずcommitを固定します。

```bash
workdir="$(mktemp -d)"
git clone https://github.com/mll-lab-nu/MindTopo.git \
  "$workdir/MindTopo"
cd "$workdir/MindTopo"
git checkout c159c95db95c425564758d6dcab8e68e4379084e

uv sync --extra openai --extra lmms-eval --extra dev
uv run python -m playwright install chromium
bash bin/setup_frontends.sh
```

remote modelを使う前に、ローカルの`oracle`または`random`でrunnerとartifact出力だけを確認できます。READMEにはAPI key不要のinterleaved smoke testがあります。

```bash
uv run python interleaved_eval_tasks/agent_runner.py \
  env=knots_untangle \
  model=oracle \
  run.manifest_limit=1 \
  run.output_dir=logs/smoke/interleaved_knots_oracle
```

確認するartifactは少なくとも次です。

- `resolved_config.yaml`：実際に解決された設定
- `model_answer.jsonl`：各stepの回答
- `summary.json`：scoreと終了状態
- `images/`または生成media：modelが見たscene

:::message
この記事では公式リポジトリのmetadata、README、設定、treeを読み、公開URLとdatasetを確認しました。リポジトリ全体の依存導入、browser runtime、remote model APIを使ったbenchmark実行は行っていません。上のコマンドは公式READMEに基づく再現手順であり、実行成功を装うものではありません。
:::

## API keyを入れる前に確認する

`.env.example`にはOpenAI、Google、NVIDIA NIMなど複数providerのkey poolと、self-hosted model向けbase URLが定義されています。評価環境を作るときは次を守ります。

1. `.env`をGitへ追加しない
2. benchmark専用で権限と上限を絞ったkeyを使う
3. provider、model ID、region、日付、commit SHAを記録する
4. request数、token、費用に上限を設定する
5. 失敗時のretryが同じepisodeを二重集計しないか確認する
6. remote APIへ送ってよい画像・promptだけを使う

READMEはcomma-separated key poolsをサポートすると説明しています。複数keyの利用はrate limit回避の手段にせず、組織の契約、利用規約、監査要件に従ってください。

## 自分のAgent評価へ移植する

MindTopoの設計は、ロボットだけでなくGUI操作や業務agentにも応用できます。

### 1. 認識とactionを別metricにする

たとえば予約画面を扱うagentなら、次を分けます。

- perception：選択中の日付、人数、料金を正しく読めたか
- state tracking：画面遷移後も条件を保持したか
- action validity：現在の画面で存在する操作を選んだか
- invariant：予算、人数、取消条件を壊していないか
- goal success：予約完了または確認待ちへ到達したか

最終的に予約できたかだけでは、偶然成功したのか、安全な手順を維持したのか区別できません。

### 2. 不変条件を機械判定する

各stepで守るべき関係を明示します。

```json
{
  "state": "confirming",
  "invariants": {
    "selected_date": "2026-09-10",
    "party_size": 2,
    "total_price_jpy_max": 20000,
    "payment_not_submitted": true
  },
  "allowed_actions": ["go_back", "request_human_review"],
  "blocked_actions": ["confirm_payment"]
}
```

自然言語の記憶だけでなく、遷移ごとにvalidatorを実行します。これはMindTopoでsimulatorが不正なactionや構造違反を判定する考え方に対応します。

### 3. 見た目を変え、構造を固定する

同じtaskについてlayout、色、object、camera、説明文を変えながら、ground truthの接続や順序は固定します。性能が急落するなら、構造ではなく表層的な手掛かりへ依存している可能性があります。

逆に見た目を似せたまま、接続や囲いだけを変えるcounterfactual pairも有効です。1 pixel単位の差ではなく、判断に必要な構造だけを制御します。

### 4. 長いepisodeほど中間状態を保存する

Planning失敗を最終scoreだけで記録せず、各stepに次を残します。

| field | 内容 |
| --- | --- |
| `observation_id` | modelが見たscene |
| `parsed_state` | modelまたはsystemが解釈した現在状態 |
| `action` | 選択した操作 |
| `action_valid` | simulator上で合法か |
| `invariants_before` | 操作前の接続・順序・境界 |
| `invariants_after` | 操作後に維持されたか |
| `goal_distance` | 目標へ近づいたか |
| `termination_reason` | success、invalid、timeout、budgetなど |

これにより、perception、state update、planning、tool executionのどこで壊れたかを切り分けられます。

## 採用前のチェックリスト

- [ ] Overall、Reasoning、Planning、task別scoreを分けて読んだ
- [ ] unweighted meanとinstance単位の集計を区別した
- [ ] model名だけでなくversionと実行日を固定した
- [ ] prompt、temperature、最大step、retry条件を保存した
- [ ] invalid action、timeout、API errorの採点方法を確認した
- [ ] simulatorのground truthとmodelの自己申告を分けた
- [ ] 見た目を変えても構造を理解できるか確認した
- [ ] 手続き生成から実世界へのdomain gapを評価した
- [ ] datasetのCC BY 4.0とcodeの未明示licenseを区別した
- [ ] Releaseがないためcommit SHAを固定した
- [ ] API key、費用、外部送信の境界を設定した
- [ ] 最終成功だけでなく各stepの不変条件を検証した

## まとめ

MindTopoは、VLMの空間能力を「画像について正しく答えたか」だけで終わらせず、接続、分離、順序、囲い、結び目をaction sequenceの中で維持できるかまで評価します。

公開結果では、最高Overallが54.13%に対し、人間は97.40%でした。代表的な複数モデルでPlanningはReasoningより弱く、静的sceneの認識を長期的な行動能力へそのまま読み替えられないことが分かります。

実務で持ち帰るべきポイントは3つです。

1. perception、state tracking、planning、executionを別々に測る
2. 接続・順序・権限など、途中で壊してはいけない不変条件を機械判定する
3. benchmarkの順位より、task別失敗、公開範囲、version、license、実環境との差を確認する

VLMやComputer Use Agentを評価するときは、1枚の画像への正答だけでなく、**変化する環境で何を維持し、どのactionが合法で、どのstepで構造を失ったか**まで記録することが重要です。

## 参考リンク

- [MindTopo公式プロジェクトページ](https://mind-topo.github.io/)
- [Microsoft Research: MindTopo reveals VLMs’ spatial reasoning abilities](https://www.microsoft.com/en-us/research/blog/mindtopo-reveals-vlms-spatial-reasoning-abilities/)
- [MindTopo公式GitHubリポジトリ](https://github.com/mll-lab-nu/MindTopo)
- [MindTopo dataset（Hugging Face）](https://huggingface.co/datasets/MLL-Lab/MindTopo)
- [GitHub Trending](https://github.com/trending?since=daily)
