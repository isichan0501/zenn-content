---
title: "Prime AgentのContinual Harnessから学ぶ安全な自己改善"
emoji: "🔁"
type: "tech"
topics: ["ai", "agent", "github", "security", "llm"]
published: true
---

2026年8月9日のGitHub Trendingでは、[Prime Agent](https://github.com/PrimeIntellect-ai/prime-agent)が首位となり、確認時点で2,483 stars todayを集めていました。Prime Agentは、コーディングや長期タスクを扱うオープンソースのエージェントです。

注目したいのは「自己改善」という言葉より、その変更対象です。Prime AgentのContinual Harnessは、ベースのシステムプロンプトを直接書き換えません。会話から得た教訓を、補助プロンプト、メモリ、スキル記述、サブエージェント仕様に分け、小さな変更として保存します。

この記事では公式README、ドキュメント、実装を基に、次を整理します。

- RLMとContinual Harnessの役割の違い
- `/refine`が変更を提案・検証・保存する流れ
- session-localとglobalを分ける理由
- 自己改善を安全に評価するための具体的な手順

:::message
GitHub Trendingとstar数は時間によって変動します。本記事の数値は2026年8月9日7時2分（JST）のスナップショットです。Trending入りは注目度の参考であり、品質や安全性の保証ではありません。
:::

## 確認した公開状況

2026年8月9日時点で確認できたリポジトリ情報は次のとおりです。

| 項目 | 確認結果 |
| --- | --- |
| リポジトリ | `PrimeIntellect-ai/prime-agent` |
| ライセンス | MIT |
| 作成日 | 2026年5月8日 |
| 最新安定版 | v0.7.1 |
| v0.7.1公開日時 | 2026年8月8日3時39分（JST） |
| 実装言語 | TypeScriptを中心とするmonorepo |
| 必要なNode.js | リポジトリの`package.json`では22.8.0以上 |

v0.7.1は、停止済みworkerの再試行と、組み込みWeb検索スキルの設定案内を修正したパッチリリースです。リポジトリは短期間に複数の0.xリリースを重ねており、更新も活発です。これは改善速度の高さを示す一方、業務へ固定導入するときはバージョンとcommit SHAを記録すべき段階でもあります。

## RLMとContinual Harnessは別の層

Prime AgentのREADMEは、中心概念を2つに分けています。

1. **Recursive Language Model（RLM）**：永続IPythonを制御面として、ファイル、シェル、スキル、子エージェントをプログラムから扱う実行モデル
2. **Continual Harness**：会話を越えて再利用したい補助プロンプト、メモリ、スキル記述、サブエージェント仕様を保存・改善する状態層

RLMは「どう実行するか」、Continual Harnessは「次のターンやセッションへ何を残すか」を主に担当します。

```text
ユーザーの目的
    ↓
ベースシステムプロンプト（変更不可）
    +
Continual Harness（変更可能な補助状態）
    ↓
RLM / 永続IPython
    ↓
ファイル・シェル・スキル・子エージェント
```

この分離は重要です。自己改善をシステムプロンプト全体の自動書換えとして実装すると、一度の誤判断で全行動へ影響が広がります。Prime Agentは変更可能な層を別にし、変更単位と影響範囲を狭めています。

## `/refine`が扱う4種類の状態

実装上、refinementの`kind`は次の4種類です。

| 種類 | 保存するもの | 例 |
| --- | --- | --- |
| `prompt` | 狭い行動方針を補う注記 | commit前に差分を読む |
| `memory` | 事実、判断、失敗、好み、結果 | このプロジェクトの公開ブランチはmain |
| `skill` | 再利用するPython呼び出しの記述 | 検証関数のimport、引数、呼び出し方 |
| `subagent` | 再利用する委任役割 | APIレビュー担当の目的と報告方法 |

ここでいうContinual Harnessの`skill`は、実行可能なパッケージそのものとは限りません。公式ドキュメントは、`/refine`が作るskill entryを、Python参照と引数契約を持つ永続的な記述と説明しています。新しい実行コードを作る場合は、別途skill creatorでパッケージ化し、読み直して検証する必要があります。

つまり、会話で見つけた手順を即座に無制限のコードへ変換するのではなく、まず「何を、どの引数で呼ぶか」という契約として残す設計です。

## 変更は提案と適用の2段階

ソースを追うと、refinementはおおむね次の順序で進みます。

1. 現在の会話、Harnessの概要、過去のrefinement履歴を集める
2. モデルへCreate / Update / DeleteからなるJSON提案を求める
3. action、kind、必須フィールド、Python参照などを検証する
4. 計画中に対象entryが変わっていないか確認する
5. 一時ファイルへ書き、renameで`harness_state.json`を置き換える
6. 変更前後を履歴へ残し、システムプロンプトを再構築する

JSON提案には、変更理由に加えて期待する結果も含まれます。実装は、モデルの出力をそのままテキスト追記するのではなく、許可された種類と操作へ絞っています。

### ベースプロンプトは変更できない

refinement用プロンプトには「base system prompt is immutable」と明記されています。検証関数も`base_system_prompt`というIDへの編集を拒否します。

これは防御として有効ですが、完全ではありません。補助prompt entryが強い指示を持てば、実際の挙動へ大きく影響できます。「ベースを変更できない」ことを「無害」と読み替えてはいけません。

### 競合した変更を拒否する

モデルが変更案を考えている間に、別セッションがglobal stateを更新する可能性があります。実装は計画開始時のbaselineと適用直前のentryを比較し、変化していれば`entry changed during refinement planning`としてその編集を適用しません。

長いLLM呼び出しを挟む処理では、この楽観的な競合検出が重要です。最後に保存した処理が他の変更を黙って上書きする事故を減らせます。

### ロールバックは変更前スナップショットから作る

各編集には`before`と`after`が記録されます。ロールバック時は適用済み編集を逆順にたどり、更新なら以前のentryへ戻し、新規作成なら削除します。

ただし、ロールバック機能があるだけでは安全とはいえません。誤ったentryが既に外部送信や削除を引き起こした場合、Harnessを戻しても副作用までは戻りません。変更後の評価を読み取り専用タスクから始める必要があります。

## localを既定にする意味

Continual Harnessには2つのscopeがあります。

- **local**：現在のセッションだけで使う
- **global**：別セッションでも使う

`await refine.run()`の既定はlocalで、`global_=True`を明示した場合だけglobalが対象です。公式のrefinement policyは、進行中タスク、短期的な障害、今回だけの調整をlocalに置き、長期的な好みや複数セッションで再利用できる手順だけをglobalへ置くよう求めています。

これは、メモリの昇格に承認段階を設ける考え方です。

```text
一度だけ観測した事象
    ↓ local
同じ失敗や成功を再観測
    ↓ 根拠と適用範囲を確認
再利用価値がある安定した規則
    ↓ explicit global
複数セッションで利用
```

一度の失敗を即座にglobal ruleへすると、特定リポジトリだけの事情が全プロジェクトへ漏れます。まずlocalで試し、再現できた教訓だけを昇格させる方が安全です。

## 長期実行で分けて考えるべき状態

Prime Agentには、Harness以外にも長期タスクを支える状態があります。

- goal：完了、停止、予算超過まで目的を保持
- autonomous mode：turn、token、時間、quality gateの上限内で継続
- compaction：古い会話を要約し、コンテキストを空ける
- heartbeat / schedule：後からセッションへ再入力する
- daemon worker：ターミナルを閉じても処理を継続する

これらは同じ「永続化」ではありません。compactionは完了判定ではなく、goalは実行権限ではなく、quality gateの成功はゲートが確認した範囲だけの証拠です。

運用時は、少なくとも次を別フィールドで記録します。

```json
{
  "objective": "対象リポジトリのテストを修正する",
  "execution_status": "awaiting_review",
  "harness_refinement": "local_only",
  "quality_gate": "tests_passed",
  "allowed_side_effects": ["edit", "test"],
  "blocked_side_effects": ["push", "publish", "delete"]
}
```

状態を分ければ、「テストが通ったので自動的に公開してよい」といった飛躍を防げます。

## ソースから再現可能に確認する

実行権限を与える前に、公式リポジトリを読み取り専用で調べられます。

```bash
workdir="$(mktemp -d)"
git clone --depth 1 https://github.com/PrimeIntellect-ai/prime-agent.git "$workdir/prime-agent"
cd "$workdir/prime-agent"

git rev-parse HEAD
node -p 'require("./package.json").version'
git grep -n "base system prompt is immutable"
git grep -n "entry changed during refinement planning"
git grep -n "harness_state.json"
```

次に、依存関係を取得してリポジトリのbuildとrefinement testを実行します。

```bash
npm ci
npm run build
npx vitest run packages/coding-agent/test/refinement.test.ts
```

この手順はモデルを起動せず、対象commitの実装とテストを確認します。記事執筆時は、commit `a18809e00ea30638584d87b3afea7285a9d7296c`でbuildとrefinement testを実行して結果を確認しました。

## 実際に試す前の安全チェック

Prime Agentの公式READMEは、workerとIPython kernelがライフサイクル分離や障害回復には役立つ一方、**セキュリティサンドボックスではない**と警告しています。モデル生成Pythonとプロジェクトコマンドは、通常は利用者と同じOS権限で動きます。

評価環境では次の順番を推奨します。

1. 安定版のtagまたはcommit SHAを固定する
2. 秘密情報を置かない使い捨てリポジトリを使う
3. 第三者skillとextensionを読み、不要なら読み込まない
4. 最初はread、edit、testだけを許可する
5. push、公開、削除、外部送信を人間承認にする
6. local refinementだけで同じ失敗が減るか測る
7. entryの根拠、対象scope、期待結果をレビューする
8. 問題があればHarnessを戻し、外部副作用も別途確認する

インストーラーは公開パッケージと`SHA256SUMS`を取得し、`sha256sum`または`shasum -a 256`で照合する実装です。それでも、リモートスクリプトを直接シェルへ渡す前に内容と取得元を確認し、バージョンを固定する方が追跡しやすくなります。

## 評価指標は「賢くなったか」だけにしない

自己改善機能の評価では、成功率だけを見ると危険な近道を見落とします。次の指標をセットで記録します。

| 観点 | 測るもの |
| --- | --- |
| 正確性 | 同じ種類のタスクで失敗が減ったか |
| 根拠 | entryが実際の会話・テスト結果に基づくか |
| 汎化 | 特定ケースへの過剰適合がないか |
| scope | localで十分な情報をglobalへ広げていないか |
| 副作用 | 不要な書込み、送信、削除が増えていないか |
| 可逆性 | 変更理由とbefore / afterを追跡し、戻せるか |
| コスト | 追加のモデル呼び出し、token、待ち時間が妥当か |

A/B比較を行うなら、同じタスク集合に対して「refinement前」「local refinement後」「global候補レビュー後」を分けます。1回の成功だけでglobalへ昇格せず、複数の独立タスクで効果と副作用を確認します。

## まとめ

Prime AgentのContinual Harnessから学べるのは、自己改善を「モデルに自由に自分を書き換えさせること」と考えない設計です。

- ベースシステムプロンプトと変更可能な補助状態を分ける
- prompt、memory、skill、subagentへ変更単位を分解する
- 既定をsession-localにして影響範囲を絞る
- JSON schema、競合検出、履歴、ロールバックを持つ
- 実行権限と学習した状態を別々に管理する
- 読み取り専用タスクから効果と副作用を測る

一方、モデル生成コードがOS権限で動く以上、Continual Harnessはサンドボックスの代わりにはなりません。採用時は「何を学んだか」だけでなく、「どこへ保存され、誰に影響し、どの証拠で昇格し、外部副作用をどう止めるか」まで設計する必要があります。

## 参考リンク

- [Prime Agent公式リポジトリ](https://github.com/PrimeIntellect-ai/prime-agent)
- [Prime Agent v0.7.1 Release](https://github.com/PrimeIntellect-ai/prime-agent/releases/tag/v0.7.1)
- [Architecture Overview](https://github.com/PrimeIntellect-ai/prime-agent/blob/main/packages/coding-agent/docs/architecture.md)
- [RLM Programming Model](https://github.com/PrimeIntellect-ai/prime-agent/blob/main/packages/coding-agent/docs/rlm.md)
- [Long-Running and Background Agents](https://github.com/PrimeIntellect-ai/prime-agent/blob/main/packages/coding-agent/docs/long-running-agents.md)
- [Refinement implementation](https://github.com/PrimeIntellect-ai/prime-agent/blob/main/packages/coding-agent/src/core/refinement/refinement.ts)
- [GitHub Trending](https://github.com/trending?since=daily)
