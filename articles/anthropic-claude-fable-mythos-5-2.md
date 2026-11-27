---
title: "AnthropicがClaude Fable 5.2とMythos 5.2をリリース、安全性と能力のトレードオフを明確化"
emoji: "📜"
type: "tech"
topics: ["ai", "anthropic", "large-model", "safety", "coding"]
published: true
---

Anthropicは2026年11月9日、新しいフロントラインモデル2機種、Claude Fable 5.2とClaude Mythos 5.2を発表した。これは同社が潜在的IPOに向準備を進めるタイミングでリリースされ、同社がAI開発のスピードを緩めるよう呼びかけてから初のモデルリリースとなった。

## モデルの概要と方向性

Anthropicによると、両モデルは同じ underlying AI model に基づいているが、安全制限とターゲットオーディエンスが異なる。

- **Claude Fable 5.2**:  potentially dangerous cybersecurity and biological research applications を制限するセーフティガードが組み込まれており、より広いユーザー層に提供される。コード作成、推論、ビジョンタスクに対応し、FrontierCode 2.0ベンチマークで49.1%のスコアを記録した。

- **Claude Mythos 5.2**: 少ない制限しか持たず、サイバーセキュリティとバイオメディカル研究を行う検証済み組織向けに提供される。Fable 5.2より制約が少なく、野心的な技術プロジェクト向けに設計されている。

## 性能評価

同社は両モデルの改善を主張している。

- **速度**: 前世代比で出力を30%速く生成。
- **コスト**: 入力トークンあたり$8/百万、出力トークンあたり$40/百万（Fable/Mythos 5.1比で20%安価）。

FrontierCode 2.0（エージェント的ソフトウェア開発ベンチマーク）でのスコア:

| モデル | スコア |
|--------|-------|
| Claude Fable 5.2 | 49.1% |
| Claude Opus 5.5 | 45.5% |
| OpenAI GPT-6 Astra | 42.8% |

FrontierSWE v2（ソフトウェア工学ベンチマーク）でのスコア:

| モデル | スコア |
|--------|-------|
| Claude Fable 5.2 | 69.9% |
| Claude Opus 5.5 | 62.3% |

デモでは、1,000以上のAIエージェントを8日間動作させ、動作するオペレーティングシステムを構築した。また、フェルマの最終定理の証明を30時間で形式化し、ブラックホールに関する理論物理学の2つの未解決問題に進展をもたらしたという。

## 利用可能性とアクセス

- **Claude Fable 5.2**: 現在、Anthropicのプラットフォーム、Claude、Claude Code、APIを通じて利用可能。Amazon Web Services、Google Cloud、Microsoft Azure でもアクセス可能。

- **Claude Mythos 5.2**: 一般提供されておらず、サイバーセキュリティとバイオメディカル研究を行う検証済み組織に限定されている。アクセスリクエストはAnthropicのウェブサイトから行える。

## 実装と導入の実際的考慮

同社は、特にMythos 5.2について、以下の点に注意するよう推奨している。

- 安全制限が緩やかであるため、組織のセキュリティポリシーと整合性があるか確認する。
- モデルは「 ambitious, open-ended technical projects」向けとされるが、成果については検証済みのソースで裏付けが必要。
- API利用時は、プロンプトインジェクション、誤ったツール選択、権限外アクセスといったリスクに対するガードレールを別途設計すること。

## まとめ

Claude Fable 5.2とMythos 5.2は、安全性と能力のトレードオフを明確に分けた2つのモデルリリースとして位置づけられる。Fable 5.2は広範なユーザーに提供され、Mythos 5.2は厳選された組織向けに提供される。どちらのモデルも、実際の成果については一次情報や検証済みのリソースに基づいて評価することを推奨する。モデルのリリースは、AI開発のペースと安全性のバランスに関する継続的な議論の最新の表明でもある。

[^1]: Anthropic. "Claude Fable 5.2 and Mythos 5.2 release announcement." *Anthropic Blog*, November 9, 2026.
[^2]: Anthropic. "FrontierCode 2.0 and FrontierSWE v2 evaluation results." *ArXiv*, 2026.
[^3]: Amodei, D. "AI development should be slowed down." *The New York Times*, November 2026.