---
title: "OpenAI Daybreakを安全に使うための権限設計"
emoji: "🌅"
type: "tech"
topics: ["openai", "ai", "security", "codex", "agent"]
published: false
---

2026年8月10日、OpenAIはサイバー防御プログラム「Daybreak」を拡張し、BlueとRedの2段階のアクセス、サイバーセキュリティ特化モデルGPT‑5.6‑Cyber、CodexのAuto-reviewを発表しました。

重要なのは、単に「高性能なセキュリティモデルが増えた」ことではありません。通常なら拒否される可能性があるデュアルユース作業を承認済みの防御側へ開放する一方で、本人確認、利用範囲、サンドボックス、監視、権限レビューを組み合わせる設計です。

この記事ではOpenAIの公式発表とCodex公式ドキュメントを基に、次を整理します。

- Daybreak BlueとRedの違い
- 公表された評価結果をどう読むべきか
- Auto-reviewが守る範囲と守らない範囲
- 組織で導入する前に固定すべき権限境界

:::message alert
本記事は防御目的の権限設計を扱います。侵入、認証回避、資格情報取得などの手順は扱いません。対象システムの所有者または管理者から、書面で許可された範囲だけをテストしてください。
:::

## Daybreakは「モデル」ではなくアクセス制度

Daybreakは、モデル名だけを指すものではありません。公式ページは、フロンティア級のサイバーモデル、Codex Security、承認済みワークフロー、パートナー企業を組み合わせた防御プログラムと説明しています。

2026年8月10日の発表では、アクセスが2段階に分けられました。

| 区分 | 主なモデル・用途 | 想定される利用者 |
| --- | --- | --- |
| Daybreak Blue | GPT‑5.6 Solなど。脆弱性発見、セキュアコードレビュー、マルウェア分析、インシデント対応、パッチ検証 | 多くの防御担当者の開始点 |
| Daybreak Red | GPT‑5.6‑Cyberなど。承認済みの高度な脆弱性研究、exploit validation、ペネトレーションテスト、red teaming | より限定・統制された専門作業 |

Blueでは、正当な防御作業を妨げることがあるシステムレベルのサイバーガードレールを外します。ただしGPT‑5.6 Sol自体が、特にデュアルユース性の高い要求を拒否する場合は残ります。

Redで利用できるGPT‑5.6‑Cyberは、GPT‑5.6 Solを基に、ゼロデイ発見やexploit chain開発など一部の専門作業を改善し、高リスクなデュアルユース要求への拒否を減らすよう訓練されています。

つまりBlueとRedの違いは、単純な性能ランクではありません。**許容する作業の危険度と、それに対応する審査・統制の強さが違う**と考えるべきです。

## 公表値は「完成率」であって正しさではない

OpenAIは、Advanced Cybersecurity Completion Rateという内部評価を公開しました。exploit chain、認証回避、権限昇格などの要求へモデルが回答を完了する割合です。

| 条件 | Completion Rate |
| --- | ---: |
| GPT‑5.6 Sol（通常） | 1.5% |
| GPT‑5.6 Sol（Daybreak Blue） | 2.0% |
| GPT‑5.5‑Cyber（Daybreak Red） | 57.3% |
| GPT‑5.6‑Cyber（Daybreak Red） | 95.0% |

この95.0%を「95%の脆弱性を発見できる」「95%正しい」と読んではいけません。測っているのは、対象要求を拒否せず完了する割合です。生成物が正しいか、安全か、対象範囲内か、修正が副作用を起こさないかは別の評価になります。

公式発表自身も、専門化のトレードオフを示しています。

- 既知脆弱性から隔離環境で動くexploitを作るExploitGym 2ではGPT‑5.6‑Cyberが改善した
- V8 exploitを扱うExploitBenchの標準300 turn条件では、GPT‑5.6 Solの方がtoken効率と成績で優位だった
- Vulnerability Discovery and Report Writingでは、GPT‑5.6‑Cyberが短く詳細不足の報告を作る場合があり、GPT‑5.6 Solを下回った

「Cyberという名前だから、すべてのセキュリティ作業で最良」とは限りません。発見、再現、深刻度評価、修正、レポートを別工程として測る必要があります。

:::message
ExploitGym 2、ExploitBench、Vulnerability Discovery and Report Writing、Completion Rateは、今回の発表ではOpenAIの内部実装または内部データを含みます。公開された説明だけでは、第三者が同じ数値を完全再現できません。
:::

## 未公開情報も採用判断に含める

GPT‑5.6‑Cyberについて、OpenAIはPreparedness Framework上でサイバー能力がHighに達するがCritical未満と評価しています。一方、詳細なsystem cardは「後日公開」とされています。

また8月7日の別発表では、今後のモデルAstraについて、予備評価の段階でCritical capabilityを排除できないと説明しました。AstraはGPT‑5.6‑Cyberとは別の今後のモデルです。この2つを混同してはいけません。

導入審査では、公開済みの性能値だけでなく、次の未確定事項も記録します。

- GPT‑5.6‑Cyberの詳細system cardが未公開
- 一部評価は内部実装で、完全な第三者再現ができない
- モデル固有の誤検知・見逃し・過剰な深刻度判定が不明
- 高度な能力を持つ今後のモデルに合わせ、統制自体が更新される可能性がある

「情報がない」をゼロリスクとして扱わず、アクセス範囲を狭くする理由として扱います。

## Auto-reviewは権限付与ではなくレビュー担当の交代

Daybreak向けのCodex運用でOpenAIが推奨しているのがAuto-reviewです。公式ドキュメントは、これを明確に「reviewer swap, not a permission grant」と説明しています。

通常は、Codexがサンドボックス境界を越える操作を要求すると、人間の承認を待ちます。Auto-reviewでは、その承認判断を別のreviewer agentへ送ります。

```text
メインエージェント
  ↓ read-only / workspace内で通常実行
境界を越える操作を要求
  ↓
Auto-review agentが要求と証拠を評価
  ├─ 許可 → 実行
  └─ 拒否 → より安全な方法へ変更、または停止
```

レビュー対象には、サンドボックス外のshell実行、許可されていないネットワーク、書込みroot外の編集、承認が必要なMCP・app toolなどが含まれます。

一方、次はAuto-reviewだけでは守れません。

1. サンドボックス内ですでに許可された操作
2. 広すぎるpermission profileで最初から到達できるファイルや宛先
3. `approval_policy = "never"`で承認要求自体が発生しない構成
4. Full Accessや`--yolo`で境界を外した構成
5. reviewer agent自身の誤判断
6. Computer Useの別系統のapp approval

Auto-reviewを有効にしてからFull Accessを与えるのではなく、**狭い境界を先に作り、その例外だけをレビューさせる**順序が必要です。

## 組織側でFull Accessを選べなくする

Codex 0.138.0以降のmanaged configurationでは、管理者が利用可能なpermission profileをallowlistにできます。次は、`read-only`と`workspace`だけを許可し、Auto-reviewを必須にする最小例です。

```toml
allowed_approval_policies = ["on-request"]
allowed_approvals_reviewers = ["auto_review"]
default_permissions = ":workspace"

[allowed_permission_profiles]
":read-only" = true
":workspace" = true
# :danger-full-access は未掲載なので拒否される
```

このTOMLはPython標準ライブラリの`tomllib`で構文確認できます。

```bash
python3 - <<'PY'
import tomllib
from pathlib import Path

path = Path("requirements.toml")
with path.open("rb") as file:
    config = tomllib.load(file)

assert config["allowed_approval_policies"] == ["on-request"]
assert config["allowed_approvals_reviewers"] == ["auto_review"]
assert ":danger-full-access" not in config["allowed_permission_profiles"]
print("managed policy syntax: OK")
PY
```

この設定はモデル利用権を付与するものではありません。ローカルクライアントの実行制約です。また、`allowed_permission_profiles`はCodex 0.138.0より前のクライアントでは無視されるため、全端末の版を確認してから配布します。

## ローカルpermission profileを狭くする

個人・検証環境でも、少なくとも次を分離します。

- 書込み可能なworkspace
- 読み取りを拒否する秘密ファイル
- 接続可能なネットワーク宛先
- 実行してよいコマンド
- 人間の承認へ戻す操作

公式仕様では、`:workspace`を継承しながら`.env`を読めなくできます。

```toml
default_permissions = "security-review"

[permissions.security-review]
extends = ":workspace"
description = "Write only in the active review workspace."

[permissions.security-review.filesystem]
glob_scan_max_depth = 3

[permissions.security-review.filesystem.":workspace_roots"]
"**/*.env" = "deny"

[permissions.security-review.network]
enabled = false
```

ネットワークを無効にした状態で、ローカルに置いた対象コードの静的レビュー、テスト生成、パッチ案作成から始めます。外部通信が必要な工程では、全通信を許可せず、対象ドメインを個別allowlistへ追加します。

macOSではSeatbelt、Linux/WSLではbubblewrapとseccompなど、OSごとに強制機構が異なります。設定ファイルが同じでも実効境界が同一とは限らないため、テスト端末で拒否動作を確認します。

## reviewer policyを書き換える前の注意

Auto-reviewは、組織の`guardian_policy_config`またはローカルの`[auto_review].policy`で方針を変更できます。ただし公式ドキュメントは、これらが既存ポリシーへ追記されるのではなく、現在のreviewer policyを置き換えると警告しています。

安全な順序は次のとおりです。

1. 現在の完全なreviewer policyを取得する
2. 既存の全ルールを残す
3. 承認済み対象、期間、操作を追加する
4. 未知のhost、production、資格情報、永続化、持出し、破壊操作を拒否する
5. ファイル・ネットワーク境界でも同じscopeを強制する
6. denyされるべきテストケースを先に実行する

完全な現行ポリシーを取得できない場合は、置換設定を使わない方が安全です。自然言語のreviewer policyは、filesystemやnetworkの強制境界の代わりにはなりません。

## 導入時は「発見」から「修正反映」まで測る

Daybreakの公式ページが強調するのは、脆弱性レポートだけではシステムは安全にならないという点です。業務評価も、モデルが何件見つけたかだけでは不十分です。

```text
候補を発見
  ↓ 独立した人またはツールが再現
深刻度と到達条件を確認
  ↓
最小パッチを作成
  ↓ 既存テスト + 回帰テスト
maintainer / ownerがレビュー
  ↓
修正を反映
  ↓
再検査・監視・必要なら協調開示
```

記録すべき指標は次のとおりです。

| 工程 | 指標例 |
| --- | --- |
| 発見 | 候補数、重複率、既知問題率 |
| 検証 | 再現率、誤検知率、scope違反数 |
| 深刻度 | 人間評価との差、過大・過小評価 |
| 修正 | テスト成功率、回帰、追加副作用 |
| 反映 | accepted patch数、修正までの時間 |
| 統制 | 境界越え要求、Auto-review拒否、human escalation |

Completion Rateが高くても、誤検知が大量に増え、修正が採用されなければ防御価値は上がりません。

## Daybreak利用前のチェックリスト

- [ ] 対象system、host、期間、許可操作を書面化した
- [ ] Daybreak Blueで不足する理由を記録した
- [ ] productionと実験環境を分離した
- [ ] workspace外と秘密ファイルのreadを拒否した
- [ ] networkをdeny-by-defaultにした
- [ ] Full Accessと`approval_policy = "never"`を禁止した
- [ ] Auto-reviewの拒否ケースをテストした
- [ ] モデル出力を独立した人またはtoolで再現した
- [ ] patchに回帰testとowner reviewを要求した
- [ ] model、client version、policy、tool call、結果を監査logへ残した
- [ ] 高リスク操作を人間へ戻す停止条件を決めた

## まとめ

OpenAI Daybreakの拡張は、高度なサイバー能力を一律に拒否するのではなく、承認済み防御者へ段階的に開放する試みです。

ただし、GPT‑5.6‑Cyberの95.0%という値は回答完成率であり、正確性や安全性ではありません。専門モデルが一般モデルを下回る評価もあり、詳細system cardも公開待ちです。

実務で重要なのは次の4点です。

1. BlueとRedを作業の危険度で選ぶ
2. Full Accessを許可せず、permission profileで先に境界を作る
3. Auto-reviewを境界越えの補助判断として使い、保証とは考えない
4. 発見件数ではなく、再現・修正・採用・scope遵守まで測る

高性能な防御モデルほど、モデルへの指示より先に、届くファイル、接続先、実行権限、承認者、停止条件を設計する必要があります。

## 参考リンク

- [Expanding Daybreak as the Cyber Defense Window Narrows](https://openai.com/index/expanding-daybreak-as-the-cyber-defense-window-narrows/)
- [Daybreak公式ページ](https://openai.com/daybreak/)
- [Putting frontier cyber models in more trusted hands](https://openai.com/index/putting-frontier-cyber-models-in-more-trusted-hands/)
- [Responding to the next frontier of critical cyber capabilities](https://openai.com/index/responding-next-frontier-critical-cyber-capabilities/)
- [Codex Auto-review](https://learn.chatgpt.com/docs/sandboxing/auto-review)
- [Codex Permission profiles](https://learn.chatgpt.com/docs/permissions)
- [Codex Managed configuration](https://developers.openai.com/codex/enterprise/managed-configuration/)
