---
title: "プロンプトを超えたコンテキスト・エンジニアリング入門（Agent-Skills-for-Context-Engineeringから学ぶ）"
emoji: "🧠"
type: tech
topics: ["llm", "agent", "prompt-engineering", "rag"]
published: false
---

## TL;DR
「Agent-Skills-for-Context-Engineering」は、プロンプト単体の工夫ではなく **“コンテキスト全体の設計”** でAIエージェントの性能を底上げするためのスキル集です[^repo]。
LLMの注意（Attention）は有限で、長文コンテキストでは中盤の情報が埋もれやすい（*lost-in-the-middle*）ことが知られています[^litm]。
だからこそ、段階的開示・メモリ・評価（LLM-as-a-Judge）などの設計パターンで「最小のトークンで最大のシグナル」を作る、というのが本記事の主題です。

## この記事で扱うこと
- なぜ「プロンプト」ではなく「コンテキスト」を設計するのか
- Agent-Skills-for-Context-Engineeringの全体像（カテゴリ別に把握）
- “濃い”実践に繋がる原則（ルール）
- 読者が試せる最小実装：段階的開示 / 評価 / JSONLメモリ

## なぜ今「コンテキスト・エンジニアリング」なのか
コンテキスト窓が大きくなっても、モデルがすべての情報を均等に使えるわけではありません。
TransformerはAttention機構でトークン間の関係を扱いますが[^attn]、長い入力では注意が分散し、重要情報が「埋もれる」ことが起きます。
とくに *lost-in-the-middle*（重要情報が中盤にあると性能が落ちる）現象が報告されています[^litm]。

なので「良い指示を書く」だけでなく、
- どの情報を
- どの順番で
- どれくらいの粒度で
- いつ読み込ませるか
を **設計** する必要があります。

## リポジトリ概要
対象: https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering[^repo]

このリポジトリは、AIエージェントを“実運用レベル”に持っていくための「コンテキスト設計の型」を集めたもの。
Claude CodeやCursorなどの環境で使うことを想定しつつ、プラットフォームに依存しない原則として整理されています。

:::message
「強いプロンプト」を探すより、**“強いコンテキスト供給パイプライン”** を作った方が再現性が高い。
:::

## スキルカテゴリ（5つ）で全体像を掴む
NotebookLMでリポジトリ内容を整理すると、エージェント開発のライフサイクルに沿って大きく5カテゴリに分けて理解すると掴みやすいです（※分類名称はリポジトリ側の構成に準拠）[^repo]。

### 1) Foundational Skills（基礎スキル）
「コンテキストのどこで劣化するか」を理解する領域。*lost-in-the-middle* のような劣化パターンを前提に、圧縮/配置戦略の土台を作る[^litm]。

### 2) Architectural Skills（設計スキル）
マルチエージェント連携、メモリ、ファイルシステムを使った動的コンテキスト管理など。「どういう構造でエージェントを組むか」を扱う領域。マルチエージェントの代表例としてAutoGenのような系譜がある[^autogen]。

### 3) Operational Skills（運用スキル）
品質・コスト・安定性を“運用”する領域。評価（LLM-as-a-Judge）を入れると改善が回る[^judge]。また、評価の土台になるベンチマーク設計の例としてAgentBenchがある[^agentbench]。

### 4) Development Methodology（開発手法）
アイデア→実装→デプロイまでの流れ、タスクとモデルの適合など。「作り方」を型化する領域。

### 5) Cognitive Architecture Skills（認知アーキテクチャ）
エージェントに“精神状態”を持たせる発想（例：BDI：Beliefs/Desires/Intentions）。外部情報をそのまま投げずに、信念/意図に変換して推論を安定させる。

## 重要な原則・ルール（濃いところ）
このリポジトリの思想を「運用できるルール」に落とすと、たとえば次が効きます。

- **Attention Mechanics（注意の力学）優先**：トークン節約そのものより「注意が当たる形」にする（重要情報を先頭/末尾に寄せる、中盤に埋めない）[^litm]
- **Progressive Disclosure（段階的開示）**：起動時に全部載せない。必要になった瞬間だけ詳細を読む[^repo]
- **Platform Agnostic（プラットフォーム非依存）**：特定のUI/SDKに依存しない「原則」としてスキルを残す[^repo]
- **Append-only memory（追記型メモリ）**：後から壊れにくい形式で、機械が読みやすい形で貯める（JSONL等）
- **SKILL.mdは肥大化させない**：スキル定義が長いほど、コンテキストを圧迫して本末転倒になりやすい

## 主要コンセプト（10個）
1. **コンテキスト・エンジニアリング**：入力（system/tools/history/docs）全体を注意予算として管理する規律[^repo]
2. **注意のメカニズム（Attention Mechanics）**：長文での注意分散を前提に高シグナル化する設計[^attn]
3. **Lost-in-the-middle**：中盤の重要情報が参照されにくい弱点[^litm]
4. **段階的開示（Progressive Disclosure）**：必要なときだけ詳細を読む設計（トークン節約）[^repo]
5. **BDI（Beliefs/Desires/Intentions）**：外部情報を「信念/欲求/意図」に落として推論を安定させる考え方[^repo]
6. **LLM-as-a-Judge**：LLMを評価者として使い、品質を計測・改善する[^judge]
7. **マルチエージェント・パターン**：役割分担で複雑さを扱う（例：AutoGen）[^autogen]
8. **メモリ設計**：短期/長期/グラフ等で知識を永続化し、検索で再利用する[^repo]
9. **ツール使用と推論の統合**：ReActのように「思考+行動」で外部ツールを織り込む[^react]
10. **プロダクション・グレード評価**：評価設計（例：AgentBench）で継続改善する[^agentbench]

## 読者が試せる“最小実装”3つ

### A) 段階的開示（Progressive Disclosure）
**狙い**：最初は短く、必要になったら読む。注意を散らさない。

1. `SKILLS.md` に「スキル名 + 1行説明 + トリガー（呼び出しキーワード）」だけ書く
2. ユーザー入力から「どのスキルが必要か」を決める
3. そのスキルの詳細ファイルだけを読み込んで、そのターンのコンテキストに注入する

### B) LLM-as-a-Judge（評価）
**狙い**：「作る」より先に「測る」を作る。改善を回す。

1. ルーブリック（評価基準）を決める（正確性/再現性/失敗の仕方/コストなど）
2. 評価用の“別エージェント（別プロンプト）”に、回答とルーブリックを渡す
3. 直接採点（1-10）またはペア比較で、改善点を文章で出させる[^judge]

### C) JSONLの追記型メモリ
**狙い**：履歴に埋めない。壊れにくく、検索しやすい形で残す。

1. スキーマを決める（例：`timestamp/entity/fact/source`）
2. 1行1JSONで `memory.jsonl` に追記する
3. 次回は「最新N行」または検索ヒット行だけをコンテキストへ注入

## まとめ
このリポジトリは「コンテキストを雑に増やす」のではなく、注意資源を前提に **情報の流れを設計する** 方向へ、エージェント開発を引っ張る資料です。

次は、
- 段階的開示
- 評価（judge）
- 追記型メモリ
のどれか1つを、自分のワークスペースに“最小構成”で入れてみるのが最短で効きます。

---

## 参考
[^repo]: https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering
[^attn]: https://arxiv.org/abs/1706.03762 （Attention Is All You Need）
[^litm]: https://arxiv.org/abs/2307.03172 （Lost in the Middle）
[^react]: https://arxiv.org/abs/2303.11366 （ReAct）
[^judge]: https://arxiv.org/abs/2306.05685 （Judging LLM-as-a-judge）
[^agentbench]: https://arxiv.org/abs/2308.00352 （AgentBench）
[^autogen]: https://arxiv.org/abs/2310.03714 （AutoGen）
