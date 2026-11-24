---
title: "Baidu ERNIE 5.5 Flash Preview - 2800億パラメータのオープンソース大規模言語モデル"
emoji: "🤖"
type: "tech"
topics: ["ai", "llm", "baidu", "foundation-model"]
published: true
---

## はじめに

2026年11月24日、BaiduはERNIE 5.5 Flash Previewをリリースしました。2800億総パラメータを持ち、トークンあたり14億パラメータがアクティブ化されるスパースMixture-of-Experts（MoE）アーキテクチャを採用しています。文脈長は最大100万トークンに対応し、Hugging FaceとModelScopeで重みが公開されています。Apache 2.0ライセンスの下でオープンソース提供されており、商用利用も可能です。

## キーフィーチャー

- **2800億総パラメータ、14億アクティブパラメータ/トークン**: MoE設計により、推論効率を維持しながら大規模な知識をエンコードします。
- **100万トークンコンテキストウィンドウ**: 長文ドキュメントやコードベースの分析に対応します。
- **原生 multimodal 機能**: テキスト、画像、動画の入力をサポートします。
- **vLLMとSGLangでの実行**: 一般的な推論フレームワークでのデプロイが容易です。
- **CodeArena WebDevスコア 1668**: オープンモデル中2位、 dense-infographicタスクでの自己修正ありと single-passの86.04から大幅に改善しています。

## 性能と評価

Artificial Analysisのインデックスによると、ベンチマークスコアは90点台前半で、現在のオープンモデルリーダーボードでは上位に位置します。同じトークン予算内では、1コストあたりの知能度が前世代モデルを上回ります。

Baiduは以下の分野で顕著な改善を報告しています。

- マルチモーダル推論: 画像キャプション、ビデオ解釈、光学的文書認識の精度向上
- 長文推論: 100万トークン上下文での整合性維持
- コード生成: 関数実装とデバッグ支援の正確性

## 利用開始

```bash
# Hugging Faceからダウンロード
git lfs clone https://huggingface.co/baidu/ernie-5.5-flash-preview

# モデルの読み込み (vLLM使用例)
from vllm import LLM
model = LLM(model="baidu/ernie-5.5-flash-preview", trust_remote_code=True)
output = model.generate("BaiduのERNIEモデルについて説明してください。", max_tokens=512)
print(output[0]["text"])
```

```python
# SGLangでの推論例
import sglang as sgl
from sglang import AssistantType

sgl.set_default_model("ernie-5.5-flash-preview")
with sgl.dialog() as dialog:
    sgl.user("このモデルの最大の強みはなんですか？")
    sgl.assistant()
print(dialog.user, dialog.assistant)
```

## ライセンスと注意事項

Apache 2.0の下で提供されており、商用利用、修正、再配布が許可されています。ただし、以下の点に注意してください。

- モデルの重みはBaiduによって管理されており、コミュニティ貢献のコードとは別です。
- ベンチマーク結果は特定のハードウェアおよび設定で測定されたものであり、全般的な性能保証ではありません。
- マルチモーダル入力は画像と動画に限定され、音声入力はサポートされていません。

## 結んで

ERNIE 5.5 Flash Previewは、オープンソースの大規模言語モデル分野におけるBaiduの最新貢献です。2800億パラメータ、100万トークンコンテキスト、multimodal機能を備え、Apache 2.0ライセンスの下で公開されています。研究者や開発者は、Hugging FaceとModelScopeから重みを入手し、vLLMやSGLangでの実験を開始できます。

参照元:
- [Baiduプレスリリース (2026年11月23日)](https://www.antgroup.com/en/news-media/press-releases/1795402800000) （Ant GroupのAIアプリAfuによるオープンソース医療モデル発表と同時期）
- [Hugging Faceモデルカード](https://huggingface.co/baidu/ernie-5.5-flash-preview)
- [ModelScopeモデルページ](https://modelscope.cn/models/baidu/ernie-5.5-flash-preview)