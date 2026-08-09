---
title: "Google公式Agent Skillsを一括導入する前の監査手順"
emoji: "🔍"
type: "tech"
topics: ["ai", "agent", "googlecloud", "security", "github"]
published: true
---

2026年8月10日のGitHub Trendingでは、Google公式の[google/skills](https://github.com/google/skills)が確認時点で532 stars todayを集めていました。Google Cloud、Gemini、GKE、BigQuery、Google Adsなどの作業手順を、AIエージェント向けのAgent Skillsとして配布するリポジトリです。

結論から言うと、これは単なるプロンプト集ではありません。調査したcommitには、104個の`SKILL.md`に加えて、PythonやShellのスクリプト、TerraformやKubernetesのテンプレート、Git submoduleが含まれていました。**公式リポジトリでも、一括導入の前に「指示」「実行コード」「外部依存」「権限」を分けて監査すべきです。**

この記事では、公式仕様とリポジトリの実体を基に、次を説明します。

- Agent Skillsがエージェントへ何を追加するのか
- `google/skills`の公開状況と収録内容
- 公式validatorで確認できること、できないこと
- 固定commitを読み取り専用で監査する手順
- Google Cloud操作を任せる前に追加すべき権限境界

:::message
GitHub Trendingとstar数は時間によって変動します。本記事の数値は2026年8月10日7時ごろ（JST）のスナップショットです。Trending入りは注目度の参考であり、品質や安全性の保証ではありません。
:::

## 確認した公開状況

調査時点のGitHub APIとリポジトリから、次を確認しました。

| 項目 | 確認結果 |
| --- | --- |
| リポジトリ | `google/skills` |
| 説明 | Agent Skills for Google products and technologies |
| ライセンス | Apache-2.0 |
| 作成日時 | 2026年4月1日4時2分（JST） |
| 調査対象commit | `092e210b243601797a0fb939040be2b1288e6d39` |
| commit日時 | 2026年8月8日7時50分（JST） |
| 累計stars | 17,184 |
| 当日増加 | 532 stars today |
| Release / tag | 調査時点ではなし |
| README上の状態 | under active development |

Releaseやtagがないため、「最新版」という名前だけでは同じ内容を再現できません。評価記録には、少なくとも40桁のcommit SHAを残します。

## Agent Skillsは実行可能な手順パッケージ

[Agent Skills仕様](https://agentskills.io/specification)では、skillは最低限`SKILL.md`を持つディレクトリです。Front Matterの`name`と`description`を使ってエージェントがskillを発見し、必要になったときだけ本文を読みます。

```text
起動時
  name / descriptionだけを読む
        ↓ タスクに一致
SKILL.mdの指示を読む
        ↓ 必要な場合
scripts / references / assetsを使う
```

この段階的開示は、すべての専門知識を常にコンテキストへ入れるより効率的です。一方、skillが有効になると、本文の指示はエージェントの行動選択へ影響し、同梱スクリプトが実行される可能性があります。

したがって、skillの導入は「Markdownを読むだけ」ではありません。少なくとも次の4層を監査します。

1. **発火条件**：どの依頼でskillが選ばれるか
2. **指示**：何を必須・禁止としているか
3. **実行物**：スクリプトやテンプレートが何を変更するか
4. **権限**：実行時にどの認証情報やクラウド資源へ届くか

## リポジトリの実体を数えた結果

調査対象commitを浅くcloneし、`skills/`配下を機械的に数えました。

```text
SKILL.md     104
scripts       59
assets         46
references    290
```

59個のscript fileの内訳は、Python 47個、Shell 7個、依存関係を記した`requirements.txt` 5個でした。Shellについては、`scripts/`以外のassetにも3個あります。また、plugin用ディレクトリには16個のGit submoduleが登録されています。

つまり監査対象は`SKILL.md`だけではありません。

```text
SKILL.md
├── 手順と行動規則
├── scripts/      Python / Shell / 依存関係
├── references/   詳細な製品知識
├── assets/       YAML / Terraform / JavaScript / template
└── plugins/      MCP serverを含み得るbundleとsubmodule
```

READMEは、汎用の`npx skills add google/skills`に加えて、Claude Code、Codex、Antigravity CLI向けpluginの導入方法も案内しています。pluginはSkillsだけでなくMCP serverも束ねる場合があるため、skill単体とは別の権限面を持ちます。

## 公式validatorで104 skillsを検証する

Agent Skillsの公式仕様は、参照実装`skills-ref`による検証を案内しています。調査では、`google/skills`の全skill directoryに対して公式validatorを実行しました。

```bash
workdir="$(mktemp -d)"

git clone --depth 1 \
  https://github.com/google/skills.git \
  "$workdir/google-skills"

git clone --depth 1 \
  https://github.com/agentskills/agentskills.git \
  "$workdir/agentskills"

uv run --project "$workdir/agentskills/skills-ref" python - \
  "$workdir/google-skills/skills" <<'PY'
from pathlib import Path
import sys
from skills_ref import validate

root = Path(sys.argv[1])
results = []

for skill_md in sorted(root.rglob("SKILL.md")):
    skill_dir = skill_md.parent
    problems = validate(skill_dir)
    if problems:
        results.append((skill_dir, problems))

print({
    "validated": sum(1 for _ in root.rglob("SKILL.md")),
    "skills_with_problems": len(results),
})

for skill_dir, problems in results:
    print(skill_dir.relative_to(root), problems)
PY
```

実行結果は次のとおりでした。

```text
{'validated': 104, 'skills_with_problems': 0}
```

追加で、収録されたPython fileを`compileall`、Shell fileを`bash -n`で構文検査し、どちらも構文エラーはありませんでした。

:::message alert
`skills-ref`自身のREADMEは、この参照ライブラリをdemonstration purposes onlyであり、本番利用向けではないと明記しています。また、validator合格はFront Matterや命名規則が仕様に合うことの証明です。スクリプトの意図、外部通信、クラウド副作用、出力の正確性までは証明しません。
:::

## まず固定SHAで内容を監査する

導入前のcloneでは、必ず対象SHAを記録します。

```bash
repo="$(mktemp -d)/google-skills"
git clone --depth 1 https://github.com/google/skills.git "$repo"
git -C "$repo" rev-parse HEAD
```

調査時の出力は次でした。

```text
092e210b243601797a0fb939040be2b1288e6d39
```

次に、収録物の数と種類を確認します。以下はPython標準ライブラリだけで動き、コード自体は実行しません。

```bash
python3 - "$repo" <<'PY'
from collections import Counter
from pathlib import Path
import sys

root = Path(sys.argv[1]) / "skills"
files = [path for path in root.rglob("*") if path.is_file()]

for dirname in ("scripts", "assets", "references"):
    selected = [path for path in files if dirname in path.parts]
    extensions = Counter(path.suffix or "<none>" for path in selected)
    print(dirname, len(selected), dict(extensions))

print("skills", sum(path.name == "SKILL.md" for path in files))
PY
```

導入候補を決めたら、対象directoryだけを読みます。

```bash
skill="$repo/skills/cloud/gcloud"

find "$skill" -type f -print
less "$skill/SKILL.md"
```

見るべき点は次のとおりです。

- `description`が広すぎて、意図しない依頼でも発火しないか
- `SKILL.md`が実行を自動承認していないか
- `scripts/`が書き込むfile、API、resourceは何か
- `requirements.txt`から追加される依存関係は何か
- `assets/`に課金や公開範囲へ影響するIaCがないか
- 参照先が公式documentか、外部の可変URLか
- 認証情報を引数、ログ、生成物へ出さないか
- destructive operationに明示承認があるか

一括導入コマンドを実行する場合も、READMEどおりの対話選択で必要なskillsだけを選びます。導入前後のSHAとfile listを保存し、更新時は差分を再監査します。

## `gcloud` skillから学べる安全策

収録された[gcloud skill](https://github.com/google/skills/blob/main/skills/cloud/gcloud/SKILL.md)は、エージェントにGoogle Cloud CLIを使わせる際の具体的なguardrailを持っています。

### 1. leaf commandのhelpを実行前に読む

`gcloud compute`のような親groupではなく、実行するleaf commandのhelpを先に確認する規則です。モデルが記憶する古いflagや、似たcommandの構文を混ぜる事故を減らします。

```text
gcloud help <実行対象のleaf command>
```

Web検索結果より、実際に導入された`gcloud`のhelpを構文の正とする点が重要です。

### 2. projectとlocationを明示する

active configurationへ暗黙に依存すると、別projectを操作する可能性があります。resourceを扱うcommandにはprojectを指定し、regionやzoneが必要ならそれも明示する方針です。

### 3. list結果を制限する

無制限のlistは、コンテキストを圧迫し、必要な行を見落とす原因になります。`--limit`、`--filter`、`--format`の少なくとも1つで結果を絞ります。

### 4. destructive operationを自動実行しない

IAM変更、APIの有効化、delete、billing、organization、KMSなどを、人間の明示承認が必要な操作として分けています。architecture skillも、生成したscriptを無断で実行しないこと、各phaseで承認を得ることを明記しています。

この設計は参考になりますが、skill内の自然言語だけを最後の安全境界にしてはいけません。実行identity側でも最小権限を設定し、重要resourceではorganization policy、IAM、network、予算alert、監査logを併用します。

## 公式であることと無条件に安全であることは別

`CONTRIBUTING.md`によると、Googleは技術的正確性、security、architectureとの整合を目的に、skillsを社内で検証・承認し、現時点では外部Pull Requestを受け付けていません。これは出所を判断する有用な情報です。

一方で、利用者側には次の責任が残ります。

- skillの想定と自社環境の差を確認する
- 対象commitと更新差分を記録する
- 実行identityを人間の管理者accountと分離する
- service account keyより短期credentialやimpersonationを優先する
- 生成commandをhelp、dry-run、planで検証する
- 課金、IAM、公開、削除を人間承認へ戻す
- 実行したcommand、対象project、結果を監査logへ残す

また、`skills.sh`の公式documentも、routine security auditを行う一方、掲載skillすべての品質やsecurityを保証できないため、利用者自身のreviewを勧めています。

## 導入を4段階に分ける

### 段階1：読むだけ

固定SHAをcloneし、`SKILL.md`、scripts、assets、submoduleを確認します。agentの設定directoryへはまだ入れません。

### 段階2：非破壊タスクで評価する

architectureの質問、公式documentの検索、commandのhelp確認など、resourceを変更しない用途で試します。出力には根拠URLと対象projectを表示させます。

### 段階3：dry-run可能な変更を試す

専用のsandbox projectと最小権限identityを用意します。Terraformなら`plan`、CLIに検証modeがある場合はそれを先に実行し、差分を人が読みます。

### 段階4：限定した変更だけを許可する

本番では、許可するservice、project、region、operationを明示します。IAM、billing、KMS、organization、delete、公開endpointの変更は、skillの指示にかかわらず別の承認gateへ置きます。

```text
GitHub repository
  ↓ SHA固定・diff review
Agent Skill
  ↓ taskごとに必要なskillだけ有効化
Agent
  ↓ 専用identity・最小権限
Google Cloud sandbox project
  ↓ plan / dry-run / human approval
限定されたresource変更
  ↓ log / operation status / rollback確認
完了
```

## 更新時は差分を再監査する

`google/skills`はactive developmentと明記され、調査直前にも更新されていました。初回だけ安全確認して終わりではありません。

```bash
git -C "$repo" fetch origin main
git -C "$repo" diff --stat HEAD..origin/main
git -C "$repo" diff HEAD..origin/main -- skills/cloud/gcloud
```

更新時は、少なくとも次を確認します。

1. `description`が変わり発火範囲が広がっていないか
2. 新しいscript、dependency、submoduleが追加されていないか
3. destructive operationの承認条件が弱くなっていないか
4. 参照documentやAPIのrelease statusが変わっていないか
5. validatorと構文検査が引き続き成功するか
6. sandbox projectで回帰testが通るか

Releaseやtagがない期間は、承認済みSHAから新しいSHAまでの差分をreview単位にします。

## まとめ

`google/skills`は、Google製品の公式知識をAgent Skillsとして再利用できる大規模なcollectionです。調査したcommitでは104 skillsを収録し、公式validatorによる仕様検証は全件成功しました。

一方で、59個のscript file、46個のasset、plugin用submoduleを含むため、validator合格や公式ownerだけで実行安全性を判断することはできません。

実務では次の順序が重要です。

1. Trendingは発見の入口として使い、品質保証とは考えない
2. commit SHA、license、更新状態、release有無を確認する
3. 必要なskillだけを選び、指示・code・dependencyを読む
4. 公式validatorと構文検査を実行する
5. 専用identity、最小権限、sandbox projectで試す
6. destructive operationは別のhuman approvalへ戻す
7. 更新差分を継続的に監査する

Agent Skillsは、エージェントへ専門知識を追加する便利な単位です。同時に、エージェントが何を信じ、何を実行するかを変更するsoftware supply chainの一部でもあります。「導入できるか」だけでなく、「どの版を、どの権限で、どの証拠を残して動かすか」まで設計して使うべきです。

## 参考リンク

- [google/skills公式リポジトリ](https://github.com/google/skills)
- [google/skills README](https://github.com/google/skills/blob/main/README.md)
- [gcloud CLI Skill for AI Agents](https://github.com/google/skills/blob/main/skills/cloud/gcloud/SKILL.md)
- [Google Cloud solution-architecture workflow](https://github.com/google/skills/blob/main/skills/cloud/google-cloud-solution-architecture/SKILL.md)
- [google/skills Contributing Policy](https://github.com/google/skills/blob/main/CONTRIBUTING.md)
- [Agent Skills Specification](https://agentskills.io/specification)
- [agentskills/agentskills公式リポジトリ](https://github.com/agentskills/agentskills)
- [skills-ref README](https://github.com/agentskills/agentskills/tree/main/skills-ref)
- [skills.sh Documentation](https://skills.sh/docs)
- [GitHub Trending](https://github.com/trending?since=daily)
