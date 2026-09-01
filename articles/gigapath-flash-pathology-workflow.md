---
title: "GigaPath-Flashで病理WSI解析を軽量化する前に知ること"
emoji: "🔬"
type: "tech"
topics: ["ai", "medical", "pytorch", "huggingface", "python"]
published: true
---

病理のWhole-Slide Image（WSI）は、1枚がgigapixel級になり、数千から数万のtileへ分割して処理します。高性能な病理Foundation Modelがあっても、tile encoderを全tileへ適用する計算量が、大規模cohortでの反復実験を難しくします。

2026年9月1日（JST）、Microsoft Researchは[GigaPath-FlashとGigaTIME-Flash](https://www.microsoft.com/en-us/research/blog/gigapath-flash-and-gigatime-flash-toward-population-scale-discovery-with-efficient-pathology-foundation-models/)の解説を公開しました。GigaPath-Flashは、22M parameterのViT-S tile encoderと21M parameterのLongNet slide encoderを組み合わせた、病理WSI向けの軽量なopen-weight modelです。

結論から言うと、GigaPath-Flashは「元のGigaPathと同じ結果を安価に出す置換」ではありません。公開論文の特定条件では約49.5倍少ないFLOPsで平均scoreの約97%を維持しましたが、評価は2つの分類benchmark、独自split、各model 1 runに限られます。研究用の特徴抽出基盤として固定版を検証し、自分のscanner・染色・施設・taskで再評価する必要があります。

この記事では、公式blog、論文v2、公開repository、Hugging Faceの配布metadataを基に、次を整理します。

- GigaPath-FlashとGigaTIME-Flashの役割の違い
- 「50倍軽量」「97%維持」を正しく読む方法
- 公開weights、実装、入力形式の確認手順
- WSI pipelineへ組み込むときに保存すべきprovenance
- 研究利用と臨床利用を分ける評価設計

:::message
2026年9月2日7時台（JST）のGitHub Daily Trendingも確認しましたが、GigaPath-Flash自体は掲載されていませんでした。本記事はTrending順位ではなく、Microsoft Researchの最新一次情報と公開artifactを根拠にしています。Trending入りやstar数は品質、安全性、臨床有用性の保証ではありません。
:::

## まず公開状況を固定する

調査時点で確認できたartifactは次のとおりです。

| 項目 | 確認結果 |
| --- | --- |
| 公式repository | `prov-gigapath/prov-gigapath` |
| 調査対象commit | `55431e04ff853a08f752a0ba42e6e9c48e60c776` |
| commit日時 | 2026年8月8日2時33分（JST） |
| release / tag | GitHub Release・tagともになし |
| code license | Apache-2.0 |
| paper | arXiv:2607.18218v2、2026年7月22日 |
| GigaPath-Flash weights | tile encoder 86.8 MB、slide encoder 86.0 MB |
| GigaTIME-Flash weights | model 95.3 MB |
| Hugging Face access | repositoryはpublicだが、weightsは利用条件への同意が必要なgated model |
| 成熟度 | early research release、臨床利用向けではない |

公式repositoryにversion tagがないため、再現時に`main`だけを記録しても不十分です。40桁のcommit SHAを固定します。

また、Apache-2.0はcodeと公開model familyの利用条件を確認する重要な情報ですが、「臨床で自由に使える」という意味ではありません。公式READMEは、checkpointを病理Foundation Model研究と論文再現向けとし、臨床判断やdeployed useを対象外にしています。

## 2つのFlash modelは出力が違う

名前は似ていますが、GigaPath-FlashとGigaTIME-Flashは同じtaskを解きません。

| Model | 入力 | 主な出力 | 主な用途 |
| --- | --- | --- | --- |
| GigaPath-Flash | H&E WSIから切り出したtileと座標 | tile embedding、文脈化されたslide representation | 分類、retrieval、下流task用の特徴抽出 |
| GigaTIME-Flash | H&E tile | 21 protein channelのvirtual mIF map | tumor microenvironmentの研究 |

### GigaPath-Flash

処理は2段階です。

```text
Whole-Slide Image
  ↓ tissue detection / tiling
224×224入力用のimage tile + WSI座標
  ↓ ViT-S/16 tile encoder（22M parameters）
384次元のtile embedding
  ↓ LongNet slide encoder（21M parameters）
slide全体の文脈を持つrepresentation
  ↓ task-specific head
分類・検索・研究用解析
```

tile encoderは、元のGigaPathにある約1B parameterのViT-g teacherから蒸留されています。slide encoderは12層、384次元のLongNetで、各tileを独立に見るだけでなく、座標と他tileの文脈を扱います。

### GigaTIME-Flash

GigaTIME-Flashは、同じViT-S系backboneを使い、H&Eからmultiplex immunofluorescence（mIF）相当の21 channelを予測します。encoderのattentionへLoRAを適用し、軽量なconvolutional decoderを組み合わせます。

これはGigaPath-Flashのslide embeddingを出す処理とは別です。「Flashのweightsを入れたらWSI分類とvirtual spatial proteomicsの両方が同じAPIで使える」と考えないでください。

## 「50倍軽量、97%維持」の分母を読む

論文の主張を実務へ持ち込む前に、測定条件を分解します。

### GigaPath-Flashの比較条件

論文Table 2は、EBRAINSで最大のslideを31,469個の256×256 px tileへ分割し、PyTorchの`FlopCounterMode`で推論FLOPsを比較しています。

| Model | TFLOPs / slide | PANDA QWK | EBRAINS balanced accuracy | 2指標の平均 |
| --- | ---: | ---: | ---: | ---: |
| GigaPath-Flash | 290.3 | 0.947 | 0.705 | 0.8260 |
| GigaPath | 14,367.3 | 0.965 | 0.741 | 0.8530 |

この条件では、GigaPathはGigaPath-Flashの49.5倍のFLOPsです。平均scoreの比は論文が約97%と報告しています。

ただし、次の範囲を越えて一般化できません。

- PANDAの前立腺grade分類とEBRAINSの脳腫瘍subtype分類だけ
- 論文独自のtrain / validation / test split
- 各model 1 runで、seedやsplitを変えた分散なし
- 共通の5 epoch training recipeで、各modelを個別最適化していない
- FLOPsは特定のtile数と実装に基づく
- survival、retrieval、治療反応予測は未評価

「平均scoreの97%」を、「診断精度97%」「全taskで性能低下3%」と読み替えてはいけません。

### GigaTIME-Flashの速度とmemory

論文はNVIDIA A100上で、batch size 128のとき次を報告しています。

- GigaTIME-Flash：peak GPU memory 2.16 GB
- GigaTIME：peak GPU memory 16.68 GB
- GigaTIME-Flash throughput：最大1,679.2 tiles / sec
- GigaTIME：およそ390 tiles / secで頭打ち

公式blogは全体を「約6倍高速、約8倍少ないmemory」と要約しています。これもA100、特定batch、特定pipelineの測定です。CPU、consumer GPU、別のCUDA・PyTorch・attention実装へそのまま適用できません。

またGigaTIME-Flashの評価指標は、8×8 pixel windowで集約したprotein activation mapのPearson correlationです。論文自身が、cell単位の正確性や臨床有用性を確立するものではないと明記しています。

## 公開weightsとcodeの境界

Hugging Face APIで確認できるGigaPath-Flashの配布物は次です。

```text
config.json
pytorch_model.bin   # tile encoder、86.8 MB
slide_encoder.pth   # slide encoder、86.0 MB
```

GigaTIME-Flashは次です。

```text
config.json
model.pth           # 95.3 MB
```

GigaPath-Flashの2つの重みは合計約164.8 MiBです。ただし、download量が小さいことと、WSI処理全体が小さいことは別です。元画像、tile、embedding、座標、中間cache、下流datasetがstorageとI/Oの主要部分になります。

さらにHugging Face上では、model repositoryはpublicでも`gated: auto`です。利用条件へ同意し、read-only tokenで取得します。tokenをsource code、notebook、shell history、記事へ埋め込まないでください。

## 固定commitを読み取り専用で検査する

最初からGPU依存を入れる前に、sourceとmanifestを確認できます。

```bash
workdir="$(mktemp -d)"

git clone https://github.com/prov-gigapath/prov-gigapath.git \
  "$workdir/prov-gigapath"

git -C "$workdir/prov-gigapath" checkout \
  55431e04ff853a08f752a0ba42e6e9c48e60c776

git -C "$workdir/prov-gigapath" rev-parse HEAD
python3 -m compileall -q "$workdir/prov-gigapath"
```

期待するSHAは次です。

```text
55431e04ff853a08f752a0ba42e6e9c48e60c776
```

記事執筆時、このcommitにあるPython 60 fileを`compileall`で検査し、exit 0を確認しました。repository内に`test_*.py`または`*_test.py`形式のtest fileはありませんでした。

:::message
`compileall`はPython構文の検査です。PyTorchやCUDA dependencyの導入、weightsのdownload、model import、GPU推論、論文benchmarkの再現を証明しません。公式test suiteがないため、導入側でsmoke testとtask固有の回帰testを作る必要があります。
:::

## 実行環境をそのまま最新化しない

固定commitの`environment.yaml`は、次を含みます。

- Python 3.9
- PyTorch 2.0.0
- torchvision 0.15.0
- CUDA 11.8
- xformers 0.0.18
- flash-attn 2.5.8
- timm 1.0.3以上
- transformers 4.36.2
- OpenSlide、MONAI、scikit-image

古い依存が含まれるからといって、すべてを最新版へ置き換えてから結果を比較すると、model差と環境差を分離できません。まず公式環境を隔離して再現し、その後に更新候補を1つずつ検証します。

```bash
conda env create -f environment.yaml
conda activate gigapath
python -m pip install -e .
```

WSI decoderやOpenSlideにはOS側libraryも必要です。GPU driver、CUDA runtime、PyTorch、xformers、flash-attnの組み合わせを記録し、container imageまたはlock済み環境を使います。

## GigaPath-Flashをloadする最小形

利用条件への同意とtoken設定後、公式READMEの入口は次です。

```python
import gigapath.slide_encoder as slide_encoder
import gigapath.tile_encoder as tile_encoder

model_id = "hf_hub:prov-gigapath/prov-gigapath-flash"

tile_enc = tile_encoder.create_model(model_id)
slide_enc = slide_encoder.create_model(
    model_id,
    "gigapath_slide_enc12l384d",
    384,
)
```

ここで重要なのは、入力次元の組み合わせです。

- tile encoder出力：384次元
- slide encoderの`input_dim`：384
- slide encoderのhidden dimension：384
- tile encoder入力：224×224へtransform
- WSI tiling：公式benchmarkでは20×、256×256 pxの非重複tileを作り、encoder入力用に変換

元のGigaPathは1536次元tile embeddingと768次元slide encoderを使います。GigaPath用に保存済みの1536次元embeddingを、そのままFlash slide encoderへ渡せません。model IDだけでなく、embedding schemaへencoder familyとdimensionを含めます。

## WSI pipelineでは座標と前処理を成果物にする

model loadより、入力を同じ意味で作る方が難しい場合があります。

最低限、次を保存します。

```json
{
  "source_slide_id": "deidentified-slide-id",
  "source_digest": "sha256:...",
  "scanner": "recorded-by-site",
  "magnification": "20x",
  "tile_size_source_px": 256,
  "tile_overlap_px": 0,
  "tissue_mask_version": "approved-pipeline-version",
  "tile_encoder": "prov-gigapath-flash",
  "tile_embedding_dim": 384,
  "slide_encoder": "gigapath_slide_enc12l384d",
  "source_commit": "55431e04ff853a08f752a0ba42e6e9c48e60c776",
  "weight_digest": "sha256:..."
}
```

患者識別子をそのままfile名やlogへ残さず、承認済みのde-identificationとaccess controlを使います。元slide、tile、embedding、label、予測、review記録を同じ識別子で追跡できるようにしつつ、研究IDと患者identityを分離します。

### cacheをmodel familyごとに分ける

```text
WSI
 ├─ preprocessing receipt
 ├─ tile image / coordinate
 ├─ GigaPath embedding（1536d）
 └─ GigaPath-Flash embedding（384d）
```

dimensionだけで判定せず、model ID、weight digest、transform、tissue mask、magnificationをcache keyへ含めます。古いembeddingを新しいmodelの出力として再利用すると、pipelineは動いても結果の意味が壊れます。

## まず行うsmoke test

weightsを取得できる環境では、実datasetへ進む前に1枚の公開・非臨床sampleで次を検査します。

1. tile encoderが有限値の384次元embeddingを返す
2. batch sizeを変えてshapeが維持される
3. 同じtileと固定環境で出力差が許容範囲内
4. slide encoderがtile embeddingと座標を受け取れる
5. tile順序と座標を同時に並べ替えても結果が安定するか確認する
6. 座標だけをずらしたnegative testで変化を検出する
7. 空slide、tissueなし、巨大slide、破損fileを成功扱いしない
8. peak memory、tiles / sec、総wall timeを自分のhardwareで測る
9. input・output・weightのdigestをrun receiptへ残す

特徴vectorが返っただけでは、病理taskとして正しいことを証明しません。次に、施設・scanner・染色・癌種・label品質を分けた外部validationが必要です。

## 下流評価を3層へ分ける

### 1. Pipeline correctness

- WSIとtileの対応が正しい
- magnification、orientation、座標系が一貫する
- backgroundとartifactを適切に除外する
- embeddingのmodel・version・dimensionを追跡できる
- 途中失敗を空の正常結果としてcacheしない

### 2. Model utility

- 自分のtaskでbaselineより有用か
- 複数seed・splitで分散を報告する
- site、scanner、染色、患者群ごとに性能を分ける
- calibration、confidence interval、failure caseを確認する
- 元GigaPathとの差を平均値だけでなく症例単位で調べる

### 3. Clinical evidence

- retrospective評価とprospective評価を区別する
- workflowへ入れたときのreader studyを行う
- 病理医の判断、処理時間、見逃し、過剰判定への影響を測る
- data driftと性能低下を監視する
- regulatory、quality management、privacy、security要件を満たす

公開論文とopen weightsが示すのは主に研究modelとしての可能性です。臨床現場で患者判断へ使える証拠ではありません。

## GigaTIME-Flashを使う場合の追加注意

virtual mIFは、実測したmIFと同じではありません。予測mapにはmodel error、registration error、施設差、染色差が含まれます。

研究では次を分けて保存します。

- H&E原画像
- GigaTIME-Flash予測map
- 利用可能な場合の実測mIF
- 21 markerの定義と前処理
- window単位の評価結果
- cohort、site、patient単位の分割
- predictionでありmeasurementではないという表示

予測値から得た仮説を、実測値で得た事実として報告してはいけません。特にpatient selection、diagnosis、prognosis、treatment choiceへ進む前に、独立したmulti-institutional・prospective validationが必要です。

## 導入前チェックリスト

- [ ] codeを40桁commit SHAで固定した
- [ ] model weightのdigestと利用条件を記録した
- [ ] GigaPathとGigaPath-Flashのembedding次元を混在させていない
- [ ] WSIのmagnification、tile size、座標、maskを保存した
- [ ] Python、PyTorch、CUDA、GPU、driverを記録した
- [ ] 公開sampleでshape、有限値、座標、error pathを検査した
- [ ] peak memoryとthroughputを自分のhardwareで測った
- [ ] 複数seed、施設、scanner、染色、患者群で評価した
- [ ] research predictionとclinical measurementを区別した
- [ ] 個人情報をfile名、cache、log、外部serviceへ漏らしていない
- [ ] 論文の49.5倍・97%を自分のSLAとして扱っていない
- [ ] 臨床判断へ使わない研究用boundaryを明示した

## まとめ

GigaPath-Flashの価値は、病理Foundation Modelを単に小さくしたことではありません。tile-levelの軽量ViT-Sとslide-levelのLongNetを組み合わせ、WSI全体の文脈を扱う研究pipelineの計算障壁を下げた点にあります。

公開論文の条件では、GigaPathに対して約49.5倍少ないFLOPsで、PANDAとEBRAINSの平均scoreの約97%を維持しました。GigaTIME-FlashもA100上で元modelより高いthroughputと低いmemory使用量を示しています。

一方で、評価taskは限定され、single runで、広い施設・scanner・患者群や臨床workflowは未検証です。安全な導入順は次です。

1. code commitとweight digestを固定する
2. preprocessingとembedding schemaを成果物として保存する
3. 公開sampleでpipeline correctnessを確認する
4. 自分の研究taskとhardwareで再benchmarkする
5. site・scanner・患者群をまたぐ外部validationを行う
6. research outputとclinical decisionの境界を維持する

「軽量になった」ことは大規模研究への入口ですが、「臨床で使える」ことの証明ではありません。計算効率、予測性能、pipeline再現性、臨床有用性を別々の証拠で評価してください。

## 参考リンク

- [Microsoft Research公式blog](https://www.microsoft.com/en-us/research/blog/gigapath-flash-and-gigatime-flash-toward-population-scale-discovery-with-efficient-pathology-foundation-models/)
- [Microsoft Research publication page](https://www.microsoft.com/en-us/research/publication/gigapath-flash-and-gigatime-flash-efficient-pathology-foundation-models-for-whole-slide-and-tumor-microenvironment-analysis/)
- [arXiv:2607.18218](https://arxiv.org/abs/2607.18218)
- [Prov-GigaPath公式repository](https://github.com/prov-gigapath/prov-gigapath)
- [調査対象commit](https://github.com/prov-gigapath/prov-gigapath/commit/55431e04ff853a08f752a0ba42e6e9c48e60c776)
- [GigaPath-Flash model](https://huggingface.co/prov-gigapath/prov-gigapath-flash)
- [GigaTIME-Flash model](https://huggingface.co/prov-gigatime/gigatime-flash)
