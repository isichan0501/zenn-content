---
title: "Google DeepMindがGemini Robotics 2.5とAlphaGenome 2を公開、ロボティクスと遺伝子解析のAIモデルを更新"
emoji: "🤖"
type: "tech"
topics: ["ai", "robotics", "deepmind", "genome", "vision-language-action"]
published: true
---

Google DeepMindは2026年11月、次世代ビジョン言語アクションモデル「Gemini Robotics 2.5」と、細胞タイプ別の遺伝子発現予測モデル「AlphaGenome 2」の公開を発表した。両モデルとも、同社の旗艦LLMGemini 4 Argonの能力拡張として位置づけられている。

## Gemini Robotics 2.5

[Google DeepMindのブログ](https://deepmind.google/blog/gemini-robotics-2-5-accelerates-real-world-physical-intelligence/)によると、Gemini Robotics 2.5は前バージョンから以下の改善を遂げている。

- **Boosted dexterity**: より正確に柔軟なオブジェクトを扱い、日常ツールの使用も可能。実証実験では自動車部品のパッキングで96%の成功率。
- **増した信頼性**: 実世界のプローフォンセプトテストでの成功率が向上。Mercedes-Benzでの実証テストで自動部品パッキング96%成功。
- **強化された安全性**:  jailbreak対抗の新ベンチマークを導入し、より安全な動作を実現。

実評価では、2.5バージョンはタスクの種類によって30%〜100%の成功率を達成し、前バージョンより20%〜30%の高速タスク完了を示した。同モデルは200以上の異なるタスクでテストされ、動的で構造化されていない実環境でのロボット導入に向けた実用性が示された。

## AlphaGenome 2

[DeepMindのブログ](https://deepmind.google/blog/alphagenome-2-deciphering-how-cells-read-dna-to-better-understand-disease/)によると、AlphaGenome 2はDNA配列とバリアエーションが人の細胞タイプに与える影響を予測するモデルで、以前のAlphaGenomeから精度と範囲が向上している。

主要な能力:

- **バリアエーションの影響予測**: 遺伝的変異が細胞の性質に与える影響を予測し、疾患関連の変異の特定を支援。
- **細胞タイプ予測**: 特定のDNA配列がどの細胞タイプに発現するかを予測。
- **ベーシックリサーチのためのツール**: ドゥーズ病やがんなどの疾患メカニズムの研究に利用可能。

AlphaGenome 2は、前モデルより広いゲノム範囲をカバーし、細胞タイプ分類の精度を向上させている。公開リソースとして、研究者は論文[^1]や評価データセット[^2]を通じてモデルを試すことができる。

## 実装と導入の実際的考慮

同社は、これらのモデルを直ちに本番導入することを推奨していない。Gemini Robotics 2.5は[^3]「PREVIEW ONLYで本番利用には適さず、APIも不安定」と明記されているとおり、プレビュー段階の実装である。AlphaGenome 2についても、バイオインフォマティクスの研究用途として実験的に利用することを推奴し、臨床的決定への直結は避けるべきとされている。

実務で導入を検討する場合、以下の点を確認することを推奨する。

1. READMEに記載の「preview only」ラベルとAPIの不安定性を理解する。
2. ネットワーク制限、実行時間、コスト上限があるかを確認する。
3. 監査ログの保存先とデータ分類方針を設計する。
4. 失敗時の安全側への停止動作を設計する。

## まとめ

Gemini Robotics 2.5とAlphaGenome 2は、それぞれロボット工学と遺伝子解析の分野で進展を遂げている。しかし、どちらも「すぐに実用化できる」段階ではなく、導入に当たっては仕様の制限と利用目的の慎重な検討が必要である。Trendingやリポジトリのスター数は注目度の指標であり、品質や安全性の保証とは断定できない。

[^1]: DeepMind. "AlphaGenome 2: Deciphering how cells read DNA." *DeepMind Blog*, 2026.
[^2]: DeepMind. "AlphaGenome 2 Evaluation Dataset." *GitHub*, 2026.
[^3]: DeepMind. "Gemini Robotics 2 Preview." *DeepMind Blog*, 2026.