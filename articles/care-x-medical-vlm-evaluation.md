---
title: "CARE-Xから学ぶ医療VLMの評価設計：生成と計測を分ける"
emoji: "🩻"
type: "tech"
topics: ["ai", "medicalai", "llm", "research"]
published: true
---

2026年8月11日、Microsoft Researchは胸部X線向けVision-Language Model（VLM）「CARE-X」の研究成果を公開しました。

注目すべき点は、レポートを流暢に生成するモデルを大きくしたことではありません。CARE-Xは自由文の生成に加えて、分類と画像上の位置を返す補助headを共同学習します。さらに別の実験では、画像から目分量で診断させず、決定論的な計測toolへ幅や比率の計算を任せています。

この記事では公式論文とMicrosoft Researchの発表を基に、次を整理します。

- 生成、構造化予測、決定論的計測を分ける理由
- 公表された評価値を再計算する方法
- 94% accuracyや100% recallからは分からないこと
- 医療以外のVLMや業務エージェントへ応用できる評価設計

:::message alert
CARE-Xは研究モデルであり、Microsoft製品でも医療機器でもありません。規制当局による承認を受けておらず、診断、screening、患者ケアを目的としていません。本記事も医療上の助言ではなく、AIシステムの設計と評価を扱います。
:::

## 公開状況を最初に確認する

2026年8月12日7時6分（JST）時点で確認できた一次情報は次のとおりです。

| 項目 | 確認結果 |
| --- | --- |
| 研究名 | CARE-X |
| 論文 | arXiv:2608.03890 |
| arXiv初回提出 | 2026年8月4日 |
| Microsoft Research発表 | 2026年8月11日 |
| 対象 | 胸部X線のreport生成、分類、VQA、grounding |
| CARE-Xのbackbone | SigLIP2-so400M + Phi-4-mini-instruct 3.8B |
| 医療機器としての状態 | 未承認の研究モデル |
| 公式code / weights | 発表・論文ページでは公開を確認できず |

したがって、現時点で第三者ができるのは、公開された論文、評価条件、集計値を読むことです。同じ重みで完全再現できる公開実装がある、すぐAPIとして利用できる、と解釈してはいけません。

## 1つの出力形式ですべてを解かない

胸部X線の読影支援には、性質の違う出力が必要です。

| 出力 | 例 | 必要な性質 |
| --- | --- | --- |
| 自由文 | findings、impression、説明 | 表現力、文脈、一貫性 |
| 分類 | 病変の有無、device配置 | 閾値、confidence、calibration |
| 位置 | 異常部位のbounding box | 座標精度、空間的一貫性 |
| 計測 | 心胸郭比、縦隔幅 | 数値精度、単位、再計算可能性 |

自由文だけに統一すると便利に見えますが、運用上の制御が難しくなります。「病変あり」と文章で答えるだけでは、感度を優先するscreeningと、偽陽性を減らしたい確認工程で閾値を変えられません。

一方、分類器だけでは、詳細な所見や質問への柔軟な回答ができません。CARE-Xは同じ表現を共有しながら、用途に応じて出力headを分けています。

```text
胸部X線
  ↓
SigLIP2 vision encoder
  ↓ adapter
Phi-4-mini-instruct
  ├─ language modeling head → report / VQA
  ├─ classification head    → P(Yes) / P(No)
  └─ grounding head         → 座標 + confidence
```

## CARE-Xの学習を4つの役割に分けて読む

論文と公式発表から、CARE-Xの構成は次のように整理できます。

### 1. 生成backbone

画像encoderのSigLIP2-so400Mと、3.8B parameterのPhi-4-mini-instructを軽量adapterで接続します。reportやVQAはautoregressiveに生成します。

### 2. 補助headによる構造化予測

分類headは病変の有無を確率として返し、grounding headは位置とconfidenceを返します。これらをlanguage modeling objectiveと共同学習します。

補助headは「生成後に別modelで検査するだけ」ではありません。共有表現へ構造化された教師信号を与え、生成側の性能にも影響させる設計です。

### 3. task別rewardによるDAPO

supervised fine-tuningの後、report、VQA、groundingごとのrewardを使ってDynamic Sampling Policy Optimization（DAPO）を行います。

重要なのは、単一の「良い回答」scoreではなく、taskごとに評価対象を変えている点です。文章の類似度、臨床的な誤り、VQAの正答、座標精度は同じ尺度では測れません。

### 4. 決定論的toolによる計測

計測実験では、CARE-Xそのものではなく、off-the-shelfのQwen3-VL-4B-Instructと決定論的な計測toolを組み合わせています。

```text
VLMが画像と質問を理解
  ↓
必要な解剖学的landmarkを指定
  ↓
toolが幅・距離・比率を計算
  ↓
VLMが画像と数値を合わせて説明
```

ここを混同してはいけません。論文の「toolで平均F1が43.6 percentage points改善」は、CARE-X単体へtoolを追加した比較ではなく、別に行われたQwen3-VL-4B-Instructの実験です。

## 閾値を変えられることが自由文との違い

Chest ImaGenome上の異常分類について、公式発表には次の値があります。

| 設定 | Sensitivity | PPV | F1 |
| --- | ---: | ---: | ---: |
| CARE-X generative | 0.932 | 0.895 | 0.913 |
| 補助head、threshold 0.5 | 0.943 | 0.885 | 0.913 |
| 補助head、threshold 0.6 | 0.855 | 0.927 | 0.890 |

thresholdを0.5から0.6へ上げると、陽性と判定する条件が厳しくなります。Sensitivityは下がり、PPVは上がっています。

つまり、補助headの価値は単にF1を最大化することではありません。同じmodelでも、見逃しを減らしたい工程と、不要な精密検査を減らしたい工程でoperating pointを選べます。

## 公表値を標準Pythonで再計算する

F1はSensitivity（Recall）とPPV（Precision）の調和平均です。丸められた公表値から次のように確認できます。

```python
rows = [
    ("generative", 0.932, 0.895, 0.913),
    ("head-th-0.5", 0.943, 0.885, 0.913),
    ("head-th-0.6", 0.855, 0.927, 0.890),
]

for name, sensitivity, ppv, reported_f1 in rows:
    calculated = 2 * sensitivity * ppv / (sensitivity + ppv)
    print(name, round(calculated, 3), reported_f1)
```

出力は次です。

```text
generative 0.913 0.913
head-th-0.5 0.913 0.913
head-th-0.6 0.89 0.89
```

記事執筆時にPython 3で実行し、表のSensitivityとPPVから求めたF1が、丸めの範囲で公表値と一致することを確認しました。

これは論文全体の再現ではありません。公開表の内部整合性を確認しただけです。元のprediction、label、confidenceがなければ、confidence interval、subgroup別性能、calibration curveは再計算できません。

## 計測toolの改善値も再計算する

別のtool-augmented実験では、5条件について次のF1が公表されています。

| 条件 | Perceptionのみ | Tool併用 | 差 |
| --- | ---: | ---: | ---: |
| Cardiomegaly | 74.56 | 96.00 | +21.44 |
| Mediastinal Widening | 72.63 | 97.47 | +24.84 |
| Aortic Knob Enlargement | 60.31 | 99.76 | +39.45 |
| Ascending Aorta Enlargement | 39.33 | 100.00 | +60.67 |
| Descending Aorta Enlargement | 28.57 | 100.00 | +71.43 |

差の単純平均は次で確認できます。

```python
pairs = [
    (74.56, 96.00),
    (72.63, 97.47),
    (60.31, 99.76),
    (39.33, 100.00),
    (28.57, 100.00),
]

deltas = [tool - perception for perception, tool in pairs]
print(sum(deltas) / len(deltas))
```

実行結果は`43.566`で、小数第1位へ丸めると公表された`43.6 percentage points`になります。

ここでも注意が必要です。この平均は5条件を同じ重みで平均した値です。症例数を重みにしたmicro averageとは限らず、実運用の有病率や損失を反映した値でもありません。

## 「計算できるものを生成させない」という設計

VLMは画像上の関係を理解できますが、pixel座標から幅を測り、比率を計算し、閾値と比較する処理まで自然言語生成へ任せる必要はありません。

たとえば心胸郭比の概念的な処理は次のように分けられます。

```python
def ratio(cardiac_width: float, thoracic_width: float) -> float:
    if cardiac_width < 0 or thoracic_width <= 0:
        raise ValueError("widths must be physically valid")
    if cardiac_width > thoracic_width:
        raise ValueError("landmarks or units must be reviewed")
    return cardiac_width / thoracic_width
```

実際の医療計測では撮影方向、拡大、体位、rotation、exposure、landmark定義などを扱う必要があり、この数行で診断はできません。それでも、設計原則は明確です。

1. VLMは何を測るべきか判断する
2. 検証可能なtoolが数値を計算する
3. 単位と入力座標を保存する
4. 閾値判定と説明を分ける
5. 元画像、overlay、数値、最終判断を追跡可能にする

これは請求書、設計図、衛星画像、品質検査などにも応用できます。数量、距離、比率、合計を「見た感じ」で生成させず、detectorや計算関数へ渡します。

## 94% accuracyをそのまま採用根拠にしない

CARE-XはReXVQAの41,007 question-answer pairsで94% overall accuracyと報告されています。論文の比較集合では次点より6 percentage points高い値です。

しかし、この数値から次は分かりません。

- 自院・自社の画像分布で同じ性能になるか
- 低画質、機器差、年齢、地域、rare conditionでどう変わるか
- confidenceが正しくcalibrateされているか
- 1件の重大な見逃しをどの程度重く扱うか
- 誤答が医療者の判断へ与えるautomation bias
- workflow全体で時間短縮やpatient outcomeが改善するか

benchmark accuracyはmodel selectionの一材料であり、deployment safetyの証明ではありません。

## 外部データ検証の読み方

Microsoft ResearchはNarayana Healthの匿名化されたretrospective dataを使った評価も報告しています。

### ICU病変の評価

1,047枚の胸部X線で、prevalence 2.6%〜5.2%の5つの高重症度条件を評価しています。CARE-Xは5条件のうち3条件で最も高いSensitivityを示しました。

一方、条件によってSpecificityは異なります。たとえば高Sensitivityだけを強調せず、false positiveがworkflowへ与える負荷も確認する必要があります。

### CT-confirmed陽性例の評価

臓器拡大については、CTで確認された122の陽性例を使い、tool-augmented variantが94.26% recallを示したと報告されています。

このstudyの主要な限界は公式発表にも明記されています。

- 評価対象は陽性例
- 目的はrecallの測定
- CT-confirmed negative cohortを含む拡張studyは進行中

陽性例だけならSensitivityは測れますが、Specificity、PPV、NPV、不要なfollow-up件数は確定できません。すべてを陽性と判定してもrecallは100%になるため、100%に近い値だけでscreening systemとして有用とは言えません。

## 導入評価ではデータを5分割する

医療に限らず、VLMを業務へ入れる前は、評価集合を次のように分けます。

1. **開発集合**：prompt、head、tool、閾値の調整
2. **固定test集合**：調整に使わない性能比較
3. **外部施設・外部分布**：domain shiftの確認
4. **時間外検証**：将来時点のdata drift確認
5. **prospective shadow運用**：実際のworkflowで結果を意思決定に使わず観測

同じtest setを見ながらthresholdやpromptを調整すると、その集合へ過剰適合します。paper上のholdoutと、自分の組織で必要な外部検証も分けます。

## 評価記録に残すべき項目

生成、分類、grounding、tool callを1つの「成功率」へ潰さず、次を保存します。

| field | 記録する内容 |
| --- | --- |
| `model_version` | 実際に実行したmodelの固定識別子 |
| `input_id` | 匿名化・仮名化された入力の追跡ID |
| `task` | report、classification、grounding、measurementなどのtask種別 |
| `view` | 確認済みの撮影方向と画像条件 |
| `model_output` | modelが生成した原文 |
| `structured_score` | 分類headが実際に返したscore |
| `threshold` | 判定に実際に使った閾値 |
| `tool_name` / `tool_version` | 計測toolと固定版 |
| `landmarks` | modelまたはdetectorが指定した座標 |
| `measurement` | 計算値、単位、計算式、validation結果 |
| `human_review` | reviewer、判定、時刻、修正理由 |

値が得られなかったfieldは架空の数値で埋めず、`null`と欠損理由を記録します。

特に、modelが出した座標とtoolが計算した値を残さないと、誤りがperception、計測、閾値、説明のどこで生じたか追跡できません。

## 実運用前のチェックリスト

- [ ] research model、product、medical deviceの状態を区別した
- [ ] code、weights、model versionを固定できるか確認した
- [ ] 自由文と構造化predictionを別々に評価した
- [ ] thresholdを用途とfalse-positive costに合わせて決めた
- [ ] calibrationとsubgroup性能を確認した
- [ ] 数値計測を生成modelだけへ任せていない
- [ ] toolの入力座標、単位、version、出力を保存した
- [ ] 外部施設・将来時点のdataで検証した
- [ ] 陽性例だけでなく陰性例を含む評価を行った
- [ ] clinicianによるoverrideと停止条件を設けた
- [ ] automation bias、遅延、障害時fallbackを評価した
- [ ] patient careへ使う前に必要な規制・倫理・data governanceを確認した

## まとめ

CARE-Xの価値は、VLMの出力をすべて自由文へ押し込まなかった点にあります。

- reportやVQAは生成headで扱う
- 感度とPPVの調整が必要な判断は分類headで扱う
- 異常位置はgrounding headで扱う
- 幅や比率は決定論的toolで計算する
- taskごとのrewardとmetricで評価する

公表値は有望ですが、CARE-Xは未承認の研究モデルであり、codeとweightsの公開も確認できません。特にCT-confirmed cohortの初期結果は陽性例中心で、Specificityを判断できません。

実務で持ち帰るべきなのは「医療VLMが94%に達した」という一文ではなく、**曖昧な知覚、構造化された判断、正確な計算、最終説明を分離し、それぞれ異なる証拠で検証する**という設計です。

## 参考リンク

- [Microsoft Research: Introducing CARE-X](https://www.microsoft.com/en-us/research/blog/introducing-care-x-towards-clinically-useful-radiology-vlms-with-auxiliary-supervision-reward-aligned-learning-and-tool-augmented-measurement/)
- [CARE-X publication page](https://www.microsoft.com/en-us/research/publication/care-x-towards-clinically-useful-radiology-vlms-with-auxiliary-supervision-reward-aligned-learning-and-tool-augmented-measurement/)
- [CARE-X paper（arXiv:2608.03890）](https://arxiv.org/abs/2608.03890)
- [ReXrank](https://rexrank.ai/)
