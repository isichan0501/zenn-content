---
title: "OpenAI Plugins移行前に行う供給網監査"
emoji: "🔌"
type: "tech"
topics: ["openai", "codex", "aiagent", "mcp", "security"]
published: true
---

2026年9月8日、OpenAI公式の[`openai/plugins`](https://github.com/openai/plugins)にQodoが追加されました。翌9日7時台（JST）のGitHub Daily Trendingでは、同repositoryが5,747 stars、176 stars todayと表示されていました。

同時にTrendingへ入っていた旧[`openai/skills`](https://github.com/openai/skills)のREADMEは、repositoryがdeprecatedであり、現在のskillとpluginの例は`openai/plugins`を使うよう案内しています。

この移行を、単なる保存先の変更として扱うべきではありません。Pluginはskillだけでなく、MCP server、browser extension、lifecycle hookをまとめられます。導入判断では「指示文を読んだ」だけでなく、実行されるcommand、接続先、認証、更新元まで確認する必要があります。

この記事では、公式repositoryの固定commitと公式documentationを基に、Pluginを業務環境へ入れる前の再現可能な監査手順を作ります。

:::message
Trendingとstar数は時間で変動します。数値は2026年9月9日7時台（JST）のsnapshotで、品質、安全性、OpenAIによる個別pluginの保証を意味しません。
:::

## まず公開状況を固定する

調査時点で確認した状態は次のとおりです。

| 項目 | 確認結果 |
| --- | --- |
| repository | [`openai/plugins`](https://github.com/openai/plugins) |
| default branch | `main` |
| 固定commit | `d416fd5a43426019986b1e489506db3db66dee3d` |
| commit日時 | 2026年9月9日2時57分（JST） |
| 最新変更 | Qodoを2つのcurated marketplaceへ追加 |
| GitHub Release | 調査時点でなし |
| GitHubが検出したrepository license | なし |
| top-level plugin directory | 62 |
| 標準catalog entry | 65 |
| API key用catalog entry | 49 |

Releaseやrepository全体のlicenseが見つからないことは、ただちに「利用不可」や「無断使用可」を意味しません。各plugin manifestには個別のlicense、利用規約、privacy policyが記載される場合があります。採用対象ごとに確認します。

まずbranchをそのままcloneして評価せず、commitを固定します。

```bash
repo_url="https://github.com/openai/plugins.git"
expected_sha="d416fd5a43426019986b1e489506db3db66dee3d"
workdir="$(mktemp -d)"

git clone --filter=blob:none "$repo_url" "$workdir/plugins"
git -C "$workdir/plugins" checkout --detach "$expected_sha"

test "$(git -C "$workdir/plugins" rev-parse HEAD)" = "$expected_sha"
git -C "$workdir/plugins" status --short
```

最後の出力が空なら、少なくとも監査開始時のtreeは固定されています。

## SkillsからPluginsへ移ると監査面が増える

旧Skillsは主に、手順、script、referenceをひとまとまりにする仕組みです。Pluginではそれに加えて外部serviceやruntime lifecycleへ接続できます。

```text
Plugin
├── manifest
├── skills/           指示、reference、補助script
├── MCP設定           外部tool・data・action
├── browser extension
└── lifecycle hooks   runtimeの特定時点でcommandを実行
```

OpenAIの公式documentationは、pluginが次の要素を含められると説明しています。

- Skills
- MCP servers
- Browser extensions
- Hooks

MCP serverは外部systemの認証と操作を担当します。HookはCodex runtimeのlifecycle eventでcommandを実行します。したがって、pluginの表示名や説明だけでは実際の権限面を判断できません。

固定snapshotを集計すると、62個のtop-level plugin directory内に次がありました。

| surface | 確認数 |
| --- | ---: |
| `.codex-plugin/plugin.json` | 64 |
| `.app.json` | 36 |
| `.mcp.json` | 31 |
| `SKILL.md` | 534 |
| `hooks.json` | 1 |
| top-level `commands/` | 6 |
| top-level `agents/` | 12 |

manifest数がtop-level directory数より多いのは、bundle内に追加のmanifestがあるためです。数だけを合格条件にせず、導入するentryから参照を辿ります。

## 互換manifestとportable manifestを混同しない

固定snapshotの例は、主に次の互換layoutです。

```text
plugin-name/
├── .codex-plugin/plugin.json
├── .mcp.json
├── .app.json
├── skills/
├── hooks.json
└── scripts/
```

一方、現在の公式build guideは、portableなAgent Plugins packageではrootの`plugin.json`とAgent Plugins schemaを使うと説明しています。OpenAI固有設定は`extensions.com.openai`へ置きます。

`.codex-plugin/plugin.json`は互換fallbackとして引き続きsupportされています。つまり、次の3点を分けて記録します。

1. repository snapshotが採用しているlayout
2. 現在のdocumentが推奨するportable layout
3. 利用するhostとversionが実際にsupportするlayout

古い例が動くことと、新規packageの推奨形式であることは同じではありません。また`.mcp.json`を単純に`mcp.json`へrenameできません。公式guideによるとportable MCP formatでは各serverのtransport `type`も宣言します。

## Marketplaceの「掲載」と実体の固定を分ける

標準catalogの65 entriesをsource種別で分けると、次の構成でした。

| source | entries |
| --- | ---: |
| local | 62 |
| URL | 2 |
| Git subdirectory | 1 |

外部sourceは次の3件です。

- CrowdStrike Falcon Foundry
- CrowdStrike Falcon Fusion
- Qodo

9月8日に追加されたQodo entryは、次のような構造です。

```json
{
  "name": "qodo",
  "source": {
    "source": "git-subdir",
    "url": "https://github.com/qodo-ai/qodo-skills.git",
    "path": "codex-packages/qodo"
  },
  "policy": {
    "installation": "AVAILABLE",
    "authentication": "ON_INSTALL"
  }
}
```

このentryにはcommit SHAやtagの指定がありません。追加commitのmessageも、upstream default branchを追従し、内容をvendoringしないと明記しています。

これは自動的に危険という意味ではありません。ただし、`openai/plugins`側のcommitだけを固定しても、後から取得するQodoの中身までは固定できません。監査時には外部sourceのSHAも別に記録します。

調査時点のQodo upstreamは次でした。

```text
repository: qodo-ai/qodo-skills
branch:     main
commit:     a16d3abbe99f1f8ab14d0772ab685fe0ca925850
license:    MIT
```

実際のinstall時に同じcommitが選ばれる保証はないため、「記事の確認結果」ではなく、自組織が取得したsnapshotを台帳へ残してください。

## catalogを機械的に監査する

次のPython scriptは、catalog entry数、source種別、refのない外部sourceを表示します。標準libraryだけで動き、pluginをinstallしません。

```python
from __future__ import annotations

import json
import sys
from collections import Counter
from pathlib import Path

root = Path(sys.argv[1]).resolve()
marketplaces = [
    root / ".agents/plugins/marketplace.json",
    root / ".agents/plugins/api_marketplace.json",
]

for path in marketplaces:
    data = json.loads(path.read_text(encoding="utf-8"))
    entries = data["plugins"]
    source_counts = Counter()
    mutable_external = []

    for entry in entries:
        source = entry.get("source", {})
        source_type = source.get("source", "unknown")
        source_counts[source_type] += 1

        is_external = source_type in {"url", "git-subdir"}
        is_pinned = any(
            key in source for key in ("ref", "sha", "commit", "version")
        )
        if is_external and not is_pinned:
            mutable_external.append({
                "name": entry["name"],
                "source": source,
                "policy": entry.get("policy"),
            })

    print(path.relative_to(root))
    print("entries:", len(entries))
    print("source types:", dict(source_counts))
    print("external sources without ref:")
    print(json.dumps(mutable_external, indent=2, ensure_ascii=False))
```

実行例です。

```bash
python3 audit_marketplace.py "$workdir/plugins"
```

固定snapshotでの要約は次のとおりでした。

```text
standard marketplace: 65 entries, 3 external sources without ref
API-key marketplace: 49 entries, 3 external sources without ref
```

ここで見ているのはcatalogの供給元です。各pluginが接続するMCP endpointや実行scriptは別の検査が必要です。

## 実行面をmanifestから辿る

### 1. Skills

`SKILL.md`は自然言語だけとは限りません。補助script、network access、file write、外部commandを指示できます。

```bash
find plugins -name SKILL.md -print
find plugins -type f -path '*/scripts/*' -print
```

全534 skillsを一括承認するのではなく、利用予定pluginのskillsだけを読みます。command例を実行する前に、download、認証情報、外部状態変更の有無を分けます。

### 2. MCP servers

固定snapshotのFigma pluginでは、`.mcp.json`がHTTP endpointを指定しています。

```json
{
  "mcpServers": {
    "figma": {
      "type": "http",
      "url": "https://mcp.figma.com/mcp",
      "oauth_resource": "https://mcp.figma.com/mcp"
    }
  }
}
```

この場合は少なくとも次を確認します。

- 接続先domainと運営主体
- OAuth scope
- read / write toolの一覧
- 送信されるprompt、file、repository情報
- service側のtermsとprivacy policy
- credentialの保存場所と失効方法
- uninstall後にも接続が残るか

OpenAIのdocumentationは、pluginをuninstallしても、別途接続したMCP integrationはChatGPT側でdisconnectするまで残ると説明しています。rollback手順には「plugin削除」と「connection切断」の両方を含めます。

### 3. Hooks

同じFigma bundleには、`Write|Edit`の後にshell scriptを起動するhookがあります。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/post_write_figma_parity_check.sh"
          }
        ]
      }
    ]
  }
}
```

公式documentationもhookを信頼してから実行するよう明記しています。最低限、次を確認します。

```bash
plugin="plugins/figma"

python3 -m json.tool "$plugin/hooks.json" >/dev/null
sed -n '1,240p' "$plugin/scripts/post_write_figma_parity_check.sh"
```

さらに、scriptが呼ぶ子command、PATHから解決されるbinary、network destination、書込先、環境変数の参照まで辿ります。相対pathがplugin rootと別のworking directoryでどう解決されるかもtestします。

:::message alert
Marketplaceへの掲載や`AVAILABLE` policyは、実行内容の包括的な安全審査結果だと読み替えないでください。承認する単位はcatalog全体ではなく、固定したplugin bundle、接続先、scope、host policyの組み合わせです。
:::

## install前に静的snapshotを作る

監査用に、対象pluginのfile listとhashを保存します。

```bash
plugin_dir="$workdir/plugins/plugins/figma"

find "$plugin_dir" -type f -print0 \
  | sort -z \
  | xargs -0 shasum -a 256 \
  > plugin-files.sha256

find "$plugin_dir" -type f -print \
  | LC_ALL=C sort \
  > plugin-files.txt
```

macOSでは`shasum -a 256`、GNU/Linuxでは`sha256sum`を利用できます。記録には次を含めます。

```text
marketplace repository + commit
external source repository + commit
plugin name + manifest version
file list + hashes
MCP endpoints + OAuth scopes
hook events + commands
required environment variables
license + terms + privacy policy
reviewer + reviewed_at
```

secret scannerやSASTを使う場合も、検出ゼロを安全証明にしません。自然言語skillによる危険な指示、正規のMCP toolによる過剰操作、hookの論理的な副作用は、credential patternだけでは見つからないためです。

## 段階的に動作確認する

### Phase 1: installしない静的検査

1. marketplaceとupstreamをSHAで固定する
2. manifest、skills、scripts、hooksを読む
3. MCP endpoint、license、termsを確認する
4. file hashとレビュー記録を残す

### Phase 2:資格情報なしの隔離環境

1. 使い捨てrepositoryまたはVMでpluginを登録する
2. environmentとfilesystem permissionを最小化する
3. MCP認証をまだ行わない
4. skill discoveryとhelpだけを確認する
5. hook発火前後のprocessとfile差分を記録する

### Phase 3: read-only credential

1. test accountを用意する
2. read-onlyまたは最小scopeで接続する
3. 取得対象外のdataへaccessできないことを確認する
4. prompt injectionを含む外部dataを入力して挙動を見る
5. logにtokenやprivate dataが残らないか確認する

### Phase 4:書込みは毎回承認

外部状態を変えるtoolは、対象、差分、宛先を表示してから承認します。OpenAIのdocumentationでも、Apple Messagesのような送信操作では、untrustedな内容を含み得るchatにper-send approvalを残すよう案内しています。

### Phase 5: rollback drill

1. pluginをuninstallする
2. MCP connectionをdisconnectする
3. OAuth tokenを失効する
4. hookが実行されないことを再確認する
5. local cache、config、logの残存を調べる
6. 導入前snapshotへ戻せることを確認する

「削除ボタンを押した」と「外部serviceへの権限がなくなった」を分けて検証します。

## 更新運用では差分を先に読む

公式CLIのbuild guideにはmarketplaceを更新するcommandがあります。

```bash
codex plugin marketplace list
codex plugin marketplace upgrade marketplace-name
```

しかし、本番環境で直接upgradeするのではなく、まず検証環境でold SHAとnew SHAの差分を取ります。

```bash
git -C "$workdir/plugins" fetch origin main
old_sha="d416fd5a43426019986b1e489506db3db66dee3d"
new_sha="$(git -C "$workdir/plugins" rev-parse origin/main)"

git -C "$workdir/plugins" diff --stat "$old_sha" "$new_sha"
git -C "$workdir/plugins" diff "$old_sha" "$new_sha" -- \
  .agents/plugins \
  plugins/figma
```

特に確認したい変更は次です。

- 新しいexternal source
- refやversionの削除
- MCP endpointの変更
- read-onlyからwrite capabilityへの変更
- hook eventやcommandの追加
- script、binary、dependencyの追加
- license、terms、privacy policyの変更

差分を承認した後に検証環境へ適用し、最後に本番へ昇格します。

## 導入判定チェックリスト

### Artifact

- [ ] marketplace repositoryを40桁SHAで固定した
- [ ] external sourceも別のSHAで固定した
- [ ] manifest versionとfile hashを保存した
- [ ] Releaseの有無と更新方式を確認した

### Execution

- [ ] すべての`SKILL.md`と補助scriptを読んだ
- [ ] hookのevent、matcher、command、子processを確認した
- [ ] browser extensionの権限を確認した
- [ ] hostのsandboxとapproval policyを確認した

### Connection

- [ ] MCP endpointと運営主体を確認した
- [ ] OAuth scopeを最小化した
- [ ] read / write actionを分類した
- [ ] 外部serviceへ送られるdataを確認した

### Governance

- [ ] plugin単位のlicenseとtermsを確認した
- [ ] test accountで段階導入した
- [ ] update前のdiff reviewを必須にした
- [ ] uninstall、disconnect、token失効を実地確認した

## まとめ

`openai/skills`から`openai/plugins`への移行で重要なのは、package名ではなく権限面が広がることです。Pluginは再利用可能なskillに加え、MCP、browser、hookを一つの配布単位にできます。

そのため、導入順序は次のようにします。

1. OpenAI側のmarketplace commitを固定する
2. vendoringされないexternal sourceも別に固定する
3. skill、MCP、hook、browserを別々に監査する
4. credentialなし、read-only、write approvalの順で試す
5. uninstallとconnection切断を分けてrollbackする

GitHub Trendingは移行を知る入口として有用でした。しかし、star増加や公式organization配下という事実だけで導入を決めず、取得したartifactと実際の権限を自分の環境で固定・検証することが重要です。

## 参照した一次情報

- [OpenAI Plugins repository](https://github.com/openai/plugins)
- [Qodo追加commit](https://github.com/openai/plugins/commit/d416fd5a43426019986b1e489506db3db66dee3d)
- [deprecatedになったOpenAI Skills repository](https://github.com/openai/skills)
- [OpenAI: Plugins](https://developers.openai.com/codex/plugins)
- [OpenAI: Package your plugin](https://developers.openai.com/codex/plugins/build)
- [OpenAI: Skills](https://developers.openai.com/codex/skills)
- [Qodo Skills repository](https://github.com/qodo-ai/qodo-skills)
- [GitHub Trending](https://github.com/trending?since=daily)
