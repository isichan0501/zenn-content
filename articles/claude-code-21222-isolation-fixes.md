---
title: "Claude Code 2.1.222の隔離修正から見直すworktree運用"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "ai", "security", "git", "agent"]
published: true
---

2026年8月4日に公開されたClaude Code v2.1.222では、worktreeで隔離したセッションやサブエージェントから、メインのチェックアウトに対して破壊的なGitコマンドを実行できた問題が修正されました。あわせて、バックグラウンドタスクで`PreToolUse`の自動許可フックがツール制限を回避できた問題も修正されています。

結論から言うと、**worktree、バックグラウンドエージェント、サブエージェント、許可フックのいずれかを使う人は、実行中のバージョンを確認し、v2.1.222以降へ更新する価値があります**。ただし、更新後もworktreeだけを完全なサンドボックスと考えてはいけません。本記事では、何が変わったかを整理し、破壊的な再現をせずに運用を点検する方法を紹介します。

:::message
本記事はAnthropicの公開リリースノートと公式ドキュメントに基づきます。問題を悪用するコマンドの再現は行っていません。
:::

## 何が変わったか

v2.1.222の変更点のうち、エージェント運用に関係する修正は主に次の4つです。

1. worktree隔離セッションと、その配下のサブエージェントからメインのチェックアウトへ破壊的なGit操作が届く問題を修正
2. ファイル編集とBashの隔離を、すべてのセッション種別へ適用
3. 要約、コンテキスト圧縮、名前変更などのバックグラウンドタスクで、`PreToolUse`自動許可フックがツール制限を回避する問題を修正
4. エージェント間の`SendMessage`を、送信前に権限分類器で評価するよう改善

さらに、リポジトリ内の`.claude/settings.json`や`.claude/settings.local.json`からRemote Controlの自動起動を有効化できなくなりました。リポジトリ側では無効化だけが可能で、有効化はユーザースコープの`/config`から行います。

これらは新しいモデル能力の追加ではなく、**隔離境界と権限判定を実装どおりに機能させるための修正**です。AIコーディングを並列化・自動化している環境ほど重要度が上がります。

## なぜworktree利用者に重要なのか

Git worktreeは、同じリポジトリから別の作業ディレクトリとブランチを作る機能です。Claude Codeでは、次のように名前付きの隔離セッションを開始できます。

```bash
claude --worktree review-auth
```

公式ドキュメントによると、既定では`.claude/worktrees/review-auth/`に作業ツリーが作られ、ブランチ名は`worktree-review-auth`になります。並列セッションのファイル編集が衝突しにくい一方、worktreeはコンテナや仮想マシンではありません。

特に重要なのは、各worktreeがリポジトリ履歴と`.git`を共有する点です。worktree内からの`git commit`を可能にするため、サンドボックスも共有Gitディレクトリへの一部書き込みを許可します。そのためClaude Codeは、単にカレントディレクトリを見るだけでなく、次のような経路を検査してメインのチェックアウトを保護します。

- `Edit`、`Write`、`NotebookEdit`がメイン側を指していないか
- Bash、PowerShell、Monitorの作業ディレクトリが隔離外へ出ていないか
- `git -C`、`--git-dir`、`GIT_DIR`、`GIT_WORK_TREE`、`cd`などでGitの操作先を変更していないか

v2.1.222は、この検査をworktreeセッション、そのサブエージェント、バックグラウンドを含む各セッション種別へ適用する修正です。つまり「別フォルダだから安全」ではなく、**共有Git状態へ到達する経路まで製品側で遮断する**ことが今回の要点です。

## 影響を受けやすい運用

`--worktree`による並列作業、`isolation: worktree`を指定したサブエージェント、バックグラウンドでの編集、`PreToolUse`による自動許可、無人のcommit・pushを利用している場合は優先して確認してください。対話セッションで読み取りだけを行う運用への直接的な影響は小さめです。

## まずバージョンと設定を確認する

### 1. 実行中のバージョンを確認する

```bash
claude --version
claude doctor
```

`claude --version`が`2.1.222`以上かを確認します。`claude doctor`はインストール状態、設定ファイルの検証エラー、直近の更新結果を読み取り専用で診断します。

ネイティブインストールはバックグラウンドで更新され、次回起動時に反映されます。すぐ更新する場合は次を実行します。

```bash
claude update
claude --version
```

npm版を使っている場合、公式手順は次のとおりです。

```bash
npm install -g @anthropic-ai/claude-code@latest
claude --version
```

Homebrewの`claude-code`は安定版チャネルで、通常は最新リリースより約1週間遅れます。公式セットアップ文書でインストール方式とチャネルを確認してください。

### 2. worktreeの状態を非破壊で確認する

```bash
git worktree list --porcelain
git status --short
git branch --show-current
```

メインと各worktreeで実行し、想定したパスとブランチにいるかを確認します。未コミット変更がある環境で、修正確認のために`reset`や`clean`を試してはいけません。隔離の確認は、捨ててもよいテスト用リポジトリで行うべきです。

### 3. 永続化された許可を見直す

Claude Code内で次を開きます。

```text
/permissions
/sandbox
```

`/permissions`では、どの設定ファイルからallow、ask、denyが入っているかを確認できます。過去に「今後は確認しない」で許可したBashルールが広すぎないか、`Bash(git *)`のような包括的な許可がないかを見直します。

## フックだけを安全境界にしない

`PreToolUse`フックはツール実行前に判定を追加できる便利な仕組みです。しかし公式ドキュメントは、コマンドフックがOSユーザーと同じ権限で動くこと、入力の検証やシェル変数のクォートが必要なことを明記しています。

今回、自動許可フックとバックグラウンドタスクの組み合わせに修正が入ったことからも、許可フックだけを唯一の防御にする設計は避けるべきです。重要な操作は通常のpermission ruleでも確認を強制します。

たとえば、次の設定は破壊的になりやすいGit操作を自動許可せず、毎回permission flowへ戻す例です。

```json
{
  "permissions": {
    "ask": [
      "Bash(git reset --hard*)",
      "Bash(git clean -fd*)",
      "Bash(git checkout --*)",
      "Bash(git push --force*)"
    ]
  }
}
```

`ask`は`allow`より先に評価されるため、別の広い許可ルールがあっても確認を要求します。完全に禁止したい組織では`deny`を検討しますが、業務上必要な復旧操作まで止める可能性があるため、対象コマンドと適用スコープを決めてから配布してください。

:::message alert
`PreToolUse`で許可されても、明示的なask・denyルールは引き続き評価されます。フック、permission rule、OSレベルのsandboxを重ねるのが基本です。
:::

## 更新後に行う安全な検証

本番リポジトリで脆弱な挙動を再現する代わりに、次を確認します。

1. `claude --version`でv2.1.222以上を確認する
2. `claude doctor`で設定エラーや更新失敗がないことを確認する
3. `/permissions`で広すぎるallowと不要な`bypassPermissions`運用を見直す
4. `/sandbox`で有効状態と依存関係を確認する
5. テスト用リポジトリで、`--worktree`セッションが専用ブランチだけを変更する通常系を試す
6. 終了後、メイン側で`git status --short`と`git log`を確認する
7. 無人ジョブは変更ファイル、commit SHA、push先を実行報告へ残す

通常系の確認では、テスト用ファイルをworktree内でcommitするだけで十分です。メイン側を狙う破壊的コマンドは不要です。なお、sandboxはBashと子プロセスをOSレベルで制限し、組み込みのRead/Edit/Writeはpermission systemが制御します。役割の違う2層として扱います。

## 注意点と制約

### worktreeは完全な実行隔離ではない

ブランチと作業ファイルの衝突回避には有効ですが、リポジトリ履歴と`.git`は共有します。信頼できないコードを実行する場合や、秘密情報・本番資格情報へ届く可能性がある場合は、dev container、コンテナ、VMなど、より強い境界も検討してください。

### sandboxの既定読み取り範囲に注意する

sandboxの既定設定は読み取り範囲が広く、認証ファイルをすべて自動拒否するわけではありません。必要に応じて`sandbox.credentials`や`denyRead`で資格情報を保護します。

### 無人実行は失敗時の停止条件を持たせる

作業ツリーが汚れている、想定外ファイルが変更された、テストが失敗した、push先が違う場合はcommitしない設計にします。隔離機能の修正と、自動化全体の安全性は別問題です。

## まとめ

Claude Code v2.1.222の重要点は、worktreeのフォルダ分離だけでなく、ファイル編集、Bash、Gitのリダイレクト、サブエージェント、バックグラウンドタスクまで隔離判定を広げたことです。並列・無人のAIコーディングを使う人ほど、早めの更新対象になります。

実務では次の3点を押さえてください。

1. v2.1.222以降へ更新し、`claude doctor`で状態を確認する
2. worktreeをコンテナと同一視せず、permissionとsandboxを併用する
3. フックの自動許可だけに頼らず、重要なGit操作はaskまたはdenyで制御する

AIエージェントの安全性は、1つの機能で完成するものではありません。隔離、権限、実行環境、差分確認、停止条件を重ね、どの層が何を守るかを明確にすることが重要です。

## 参考リンク

- [Claude Code v2.1.222 Release Notes](https://github.com/anthropics/claude-code/releases/tag/v2.1.222)
- [Run parallel sessions with worktrees](https://docs.anthropic.com/en/docs/claude-code/worktrees)
- [Configure permissions](https://docs.anthropic.com/en/docs/claude-code/permissions)
- [Configure the sandboxed Bash tool](https://docs.anthropic.com/en/docs/claude-code/sandboxing)
- [Hooks reference](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [Advanced setup and updates](https://docs.anthropic.com/en/docs/claude-code/setup)
