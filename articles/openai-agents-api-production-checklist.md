---
title: "OpenAI Agents APIを本番導入する前の評価チェックリスト"
emoji: "🧰"
type: "tech"
topics: ["openai", "aiagent", "python", "security", "api"]
published: true
---

2026年9月10日、OpenAIは長時間動作するクラウドエージェント向けの[Agents API](https://openai.com/index/introducing-the-agents-api/)をpublic betaとして公開しました。

Agents APIは、単発のモデル呼び出しにtool loopを足しただけのAPIではありません。OpenAIがCodexで使うmanaged harnessを通じて、session、context compaction、復旧、sandbox、artifact、subagentまで扱います。その代わり、アプリケーションが管理すべき状態と権限も増えます。

この記事では、公式発表、公式ドキュメント、対応するPython SDK `3.13.0`を基に、次を整理します。

- Responses APIやAgents SDKと混同しないための境界
- OpenAI-hosted / self-hosted sandboxの選び方
- 「turn completed」を業務上の成功と誤認しない監視
- network、credential、data retentionの初期確認
- 小さく評価してから本番へ進む手順

:::message
Agents APIはpublic betaです。仕様は短期間で変わる可能性があります。本記事は2026年9月13日7時台（JST）に確認したsnapshotであり、一般提供（GA）や後方互換性を保証するものではありません。
:::

## まず公開状態を固定する

調査時点で確認した公開物は次のとおりです。

| 項目 | 確認結果 |
| --- | --- |
| Agents API | public beta |
| REST endpoint | `/v1/agents/sessions` |
| beta header | `OpenAI-Beta: agents=v1` |
| Python SDK | `openai==3.13.0` |
| SDK Release | 2026年9月10日19時37分 UTC（JSTでは9月11日4時37分） |
| Release commit | `f0fa922ef12f2c7329bcd8fc42e0cbb46f008ecb` |
| PyPI wheel SHA-256 | `e35b1f6fe99245e86e37504d9fad1ad2a363807307c424232c1d849bd0666c8e` |
| SDK license | Apache-2.0 |
| data residency | 調査時点では米国のみ |
| Zero Data Retention | 非対応 |

OpenAIのPython SDK `3.13.0`のRelease notesは、Agents API追加を唯一のfeatureとして挙げています。SDKのtag、PyPI metadata、wheel内の`openai/_version.py`を照合し、wheel内に`openai/resources/beta/agents/agents.py`が含まれることも確認しました。

隔離環境でversionとAPI surfaceを調べるには、次のようにします。

```bash
env -u VIRTUAL_ENV -u PYTHONPATH \
  uv run --no-project --isolated --python 3.11 \
  --with 'openai==3.13.0' \
  python -c '
import inspect
import openai
from openai import OpenAI

client = OpenAI(api_key="type-check-only")
print(openai.__version__)
print(hasattr(client.beta, "agents"))
print(inspect.signature(client.beta.agents.sessions.create))
'
```

確認時のversionは`3.13.0`、`hasattr`は`True`でした。これはclient libraryの型とmethodが存在することの確認です。API keyに仮の値を使っており、session作成やmodel推論の成功を示す実行結果ではありません。

## 4つのresourceを分けて考える

公式overviewはAgents APIを4つの概念に分けています。

| resource | 主な役割 | アプリケーション側で記録するもの |
| --- | --- | --- |
| Agent | model、instructions、tools、MCP | 定義version、model、tool allowlist |
| Environment | file、command、skillを扱う実行環境 | environment ID、方式、network policy |
| Session | 複数turnを継続する状態 | session ID、所有者、業務status |
| Events / items | 入力、進行、tool実行、出力 | event cursor、turn ID、item ID、判定結果 |

重要なのは、**SessionとEnvironmentの寿命が同じではない**ことです。

Agents APIはsession stateを保持し、以前の作業を要約しながら複数のcontext windowをまたいで継続できます。一方、OpenAI-hosted sandboxはactivityとkeep-aliveが1時間止まると削除される可能性があります。session IDが残っていても、以前のlive filesystemが必ず残っているとは限りません。

したがって、業務DBに`session_id`だけを保存して「再開可能」と判定してはいけません。

```text
アプリケーション上のjob
├── session_id
├── environment_id
├── environment_state
├── root_turn_id
├── expected_artifacts
├── last_observed_event
└── business_status
```

再開前には、session、environment、必要なinput file、artifactの存在を別々に確認します。

## Agents API、Agents SDK、Responses APIは別の選択肢

名前が似ていますが、役割は同じではありません。

- **Responses API**: model responseとtool callを組み立てる基礎API
- **Agents SDK**: アプリケーション側でagent orchestrationを構築するopen-source SDK
- **Agents API**: OpenAIがCodex harness、session、compaction、recoveryを管理するhosted API

Agents APIを採用する理由は、「agentという名前だから」ではなく、managed harnessの責務をOpenAIへ移す価値がある場合です。

比較時には少なくとも次を測ります。

| 観点 | 自前orchestration | Agents API |
| --- | --- | --- |
| context管理 | 実装・調整が必要 | managed compaction |
| durable session | 自前storeと復旧 | APIがsessionを保持 |
| sandbox | 別途用意 | hosted / self-hostedを選択 |
| 詳細な制御 | 実装次第 | beta APIの抽象化に従う |
| data residency | 自社構成次第 | 調査時点で米国のみ |
| ZDR | 構成次第 | 調査時点で非対応 |
| 移行risk | 自社codeの変更 | beta仕様とmanaged harnessの変更 |

既存のResponses API実装を、feature一覧だけで一括置換するのは避けます。同じtask setを両方で実行し、成功率、tail latency、cost、復旧性、監査可能性を比較します。

## 最小構成はnetworkを無効にして始める

OpenAI-hosted sandboxはLinux workspaceを提供し、Python、Node.js、command-line toolsを使えます。working directoryは`/workspace`です。

注意したいのは、sandboxのoutbound networkが**既定でenabled**であることです。評価の最初は、必要性が証明されるまで`disabled`にします。

次は、入力CSVをinlineで渡し、networkなしで集計結果をartifactにする公式例を小さくしたものです。

```python
from openai import OpenAI

client = OpenAI()
stream = client.beta.agents.sessions.create(
    agent={
        "model": "gpt-6-astra",
        "instructions": (
            "入力を変更せずに集計する。"
            "計算結果をJSONへ保存し、読み戻して検証する。"
        ),
    },
    environment={
        "type": "openai_hosted",
        "network": {"access": "disabled"},
        "files": [
            {
                "type": "inline",
                "path": "/workspace/amounts.csv",
                "data": "YW1vdW50CjEwCjIwCjMwCg==",
            }
        ],
    },
    input=(
        "/workspace/amounts.csv のamount列を合計し、"
        "/workspace/outputs/summary.jsonへ保存する。"
        "保存後に読み戻し、内容を報告する。"
    ),
    stream=True,
)

with stream:
    for event in stream:
        print(event.model_dump_json())
```

このbase64は次のCSVです。

```csv
amount
10
20
30
```

`/workspace/outputs`に置いたfileは、turn完了時にimmutable artifactとして公開されます。sandboxのlive filesystemが失効してもartifact copyはdownloadできます。ただし、必要なartifactを保存してからsessionを削除します。

### networkを開くときの順序

`network.access`には3段階があります。

| 値 | 挙動 |
| --- | --- |
| `disabled` | outbound accessを遮断 |
| `restricted` | `allowed_domains`にあるhostだけ許可 |
| `enabled` | outbound accessを許可 |

`restricted`では、protocol、path、port、wildcardではなく、完全なhost名を1〜100件指定します。subdomainとredirect先も別のentryが必要です。

導入順序は次のようにします。

1. `disabled`でlocal処理だけを通す
2. access logから本当に必要な接続先を特定する
3. `restricted`でhostを個別に許可する
4. redirect先とdownload CDNも明示する
5. `enabled`が必要なら理由と期限を残す

package installを許可すると、直接の接続先だけでなくregistry、artifact storage、redirect先へ到達する可能性があります。「PyPIだけ」「npmだけ」という説明ではallowlistとして不十分です。

## credentialはsandboxへ渡さない

公式security guideは、agentがenvironment内のfile、credential、networkへaccessできると明記しています。agent-generated codeも同じ境界に入ります。

OpenAI-hosted sandboxでは、applicationの`OPENAI_API_KEY`をenvironment variableへ設定できません。self-hosted方式でも、application keyとexecutor keyを分けます。

self-hosted sandboxは次の構成です。

```text
Application
  └── Agents APIへsessionを作成

Self-hosted environment
  └── codex exec-server
        ├── restricted CODEX_API_KEY
        ├── outbound WebSocket
        └── command / file / local MCPを実行
```

executor keyはenvironment接続専用にし、他のpermissionを`None`にします。agent-generated codeから読める可能性はありますが、他のAPI actionを認可できないよう権限を狭めます。

外部service用credentialも、可能ならsandboxへ直接injectしません。credential brokerを挟み、許可した宛先と操作だけへ短期credentialを付与します。

最低限の設計は次のとおりです。

- application keyはsandbox外に置く
- executor keyはsession所有者と同じorganization / projectへ限定する
- 外部tokenはread-onlyから始める
- write actionは対象と差分を表示して承認する
- secretをsource、container image、log、artifactへ残さない
- revokeとrotateを通常手順として試す

## event streamをsuccess判定に使う

quickstartは、`agent.session.turn.completed`を待つだけでは不十分だと説明しています。

- `agent.session.idle`だけでは成功ではない
- streamが閉じただけでも成功ではない
- `turn.completed`でも、すべてのtoolが成功した保証はない
- 切断時は再実行前にsessionとsaved itemsを読み戻す

業務上のsuccessは、API eventと成果物検証を組み合わせて決めます。

```python
TERMINAL_FAILURES = {
    "agent.session.turn.failed",
    "agent.session.turn.cancelled",
    "agent.session.failed",
    "agent.session.environment.failed",
    "error",
}

root_turn_completed = False
failure = None

for event in events:
    kind = event.type

    if kind in TERMINAL_FAILURES:
        failure = event
        break

    if kind == "agent.session.turn.completed":
        turn = event.turn
        if turn.subagent_id is None:
            root_turn_completed = True
            break

if failure is not None:
    raise RuntimeError(f"agent failed: {failure}")
if not root_turn_completed:
    raise RuntimeError("root turn completion was not observed")
```

この判定後にも、次を確認します。

1. root agentの最終messageが存在する
2. 必須tool itemが成功している
3. 期待したartifact名、MIME type、sizeが一致する
4. artifactをdownloadしてschemaを検証できる
5. 外部writeがある場合は対象systemから読み戻せる
6. application側のidempotency keyまたはjob IDと対応する

stream切断後に同じtaskをすぐ再送すると、外部writeや長時間処理が重複する可能性があります。session itemsとturn statusを取得し、進行中、完了、失敗を判別してから再試行します。

## subagentは計算資源と権限面を増やす

multi-agentを有効にすると、coordinatorは独立したtaskをsubagentへ委譲できます。各subagentは別contextを持ちますが、environmentを新しく作るわけではありません。

公式ドキュメント上の重要な境界は次のとおりです。

- 同じenvironmentのfilesystemとcommand-line toolsを共有する
- MCP tools、credential、allowed toolsを継承する
- web search設定も継承する
- function toolsはsubagentではsupportされない
- defaultの同時subagent数はcoordinatorを除いて6

したがって、parallel化は最初から最大にしません。

```python
agent = {
    "model": "gpt-6-astra",
    "instructions": (
        "独立した調査だけをsubagentへ分ける。"
        "同じfileを複数agentで編集しない。"
        "根拠URLと未確認事項を分けて返す。"
    ),
    "multi_agent": {
        "enabled": True,
        "max_concurrent_subagents": 2,
    },
}
```

比較する指標はwall-clockだけではありません。

| 指標 | 確認内容 |
| --- | --- |
| correctness | 単一agentと同じ合格条件を満たすか |
| conflict | shared filesystemで上書きが起きないか |
| cost | rootと全subagentのmodel callを含むか |
| observability | commandを実行したsubagentを追跡できるか |
| permissions | 継承したMCP scopeが過剰でないか |
| recovery | 一部subagent失敗時に全体をどう判定するか |

turnの`subagent_id`が`null`ならroot agentです。command itemの`turn_id`からturnを取得すると、実行主体を追跡できます。

usageはbest-effortで、`null`の場合があり、後から変わることもあります。missing usageをzero costとして扱わず、請求情報、sandbox料金、tool料金、外部service料金と照合します。

## data retentionを採用gateにする

self-hosted sandboxを選んでも、Agents API自体がZero Data Retention対応になるわけではありません。公式overviewは、調査時点で次を明記しています。

- data residencyは米国のみ
- Zero Data Retentionは非対応
- session stateを保持する
- 不要なsessionと公開済みartifactは削除できる

これは実装後に法務確認する項目ではなく、PoC開始前のgateです。

```text
[ ] 対象dataを米国で処理・保持できる
[ ] ZDR必須のworkloadではない
[ ] sessionへ送るdata分類を決めた
[ ] artifactのretention ownerを決めた
[ ] deleteを終了処理と障害復旧の両方で試した
[ ] dashboard traceへaccessできる担当者を限定した
```

なお、詳細traceの取得は通常のproject API key向けpublic beta APIには含まれず、dashboard側で確認します。外部trace exporterがあると仮定して監査設計を作らない方が安全です。

## 段階的な評価手順

### Phase 1: SDKとschemaだけを固定する

1. SDK `3.13.0`とhashを固定する
2. `client.beta.agents`の存在を検査する
3. request modelとevent typeをtest fixtureでparseする
4. beta headerをHTTP logで確認できるようにする
5. API callなしのunit testと、APIを使うintegration testを分ける

### Phase 2: networkなし・外部toolなし

1. `environment.type: openai_hosted`
2. `network.access: disabled`
3. 人工的なinput fileだけを渡す
4. `/workspace/outputs`へschema固定のartifactを作る
5. root turn、tool item、artifact内容をすべて検証する
6. session削除とsandbox cleanupを試す

### Phase 3: read-only連携

1. 接続先を`restricted`でallowlistする
2. read-only MCPまたはbroker経由の短期credentialを使う
3. prompt injectionを含む外部dataを入力する
4. 許可外domainとwrite actionが失敗することを確認する
5. credentialがlog、artifact、model出力へ出ないことを確認する

### Phase 4: subagent比較

1. 独立した2 taskだけで有効化する
2. `max_concurrent_subagents: 2`から始める
3. 単一agentと同じtask setで比較する
4. root / subagent別のturn、command、usageを保存する
5. shared filesystemの競合をfault injectionする

### Phase 5: write actionと復旧

1. write前にhuman approvalを要求する
2. application job IDを外部systemにも保存する
3. streamを意図的に切断する
4. sessionとitemsを読み戻してから再開する
5. 同じwriteが二重実行されないことを確認する
6. timeout、409、5xxに回数上限とdeadlineを設ける

## 本番導入チェックリスト

### Version

- [ ] Agents APIがpublic betaであることを承認した
- [ ] SDK version、wheel hash、Release commitを固定した
- [ ] beta仕様変更を検知する契約testがある

### Environment

- [ ] hosted / self-hostedの選定理由がある
- [ ] workload単位でenvironmentを分離した
- [ ] networkを既定のenabledのまま放置していない
- [ ] sandbox失効後の再構築手順がある
- [ ] artifactとlive fileの寿命を分けている

### Credentials

- [ ] application keyをsandboxへ渡していない
- [ ] executor keyは接続専用である
- [ ] 外部credentialをbrokerまたは最小scopeにした
- [ ] rotateとrevokeを実地確認した

### Completion

- [ ] idleやstream closeをsuccess扱いしていない
- [ ] root turnとsubagent turnを区別している
- [ ] completed後にtoolとartifactを検証している
- [ ] 切断時に読み戻してからretryする
- [ ] 外部writeにidempotencyがある

### Governance

- [ ] 米国data residencyを許容できる
- [ ] ZDR非対応を許容できる
- [ ] sessionとartifactの削除方針がある
- [ ] dashboard traceの閲覧権限を管理している
- [ ] best-effort usageを最終請求と混同していない

## まとめ

Agents APIの価値は、Codexで使われるharnessをmanaged serviceとして利用し、長時間session、context compaction、sandbox、artifact、subagent、復旧を一つのAPI面で扱えることです。

一方、「managed」は責任がなくなることを意味しません。導入時には次を分けて検証します。

1. sessionが残ることと、sandbox fileが残ること
2. turnがcompletedになることと、tool・業務処理が成功すること
3. self-hosted computeを使うことと、ZDR要件を満たすこと
4. subagentで速くなることと、cost・権限・競合が増えること
5. credentialを環境変数へ入れられることと、安全に委譲できること

まずnetworkなし、外部toolなし、人工dataだけの1 turnから始めます。event、saved item、artifactの3つで同じ結果を確認し、削除と切断復旧まで通してからread-only連携へ進むのが安全です。

## 参照した一次情報

- [OpenAI: Introducing the Agents API](https://openai.com/index/introducing-the-agents-api/)
- [OpenAI Developers: Agents API overview](https://developers.openai.com/api/docs/guides/agents-api/overview)
- [OpenAI Developers: Agents API quickstart](https://developers.openai.com/api/docs/guides/agents-api/quickstart)
- [OpenAI Developers: OpenAI-hosted sandboxes](https://developers.openai.com/api/docs/guides/agents-api/environments/openai-hosted)
- [OpenAI Developers: Self-hosted sandboxes](https://developers.openai.com/api/docs/guides/agents-api/environments/self-hosted)
- [OpenAI Developers: Sandbox security](https://developers.openai.com/api/docs/guides/agents-api/environments/security)
- [OpenAI Developers: Events and items](https://developers.openai.com/api/docs/guides/agents-api/sessions/events)
- [OpenAI Developers: Multi-agent](https://developers.openai.com/api/docs/guides/agents-api/multi-agent)
- [OpenAI Developers: Observability and usage](https://developers.openai.com/api/docs/guides/agents-api/observability)
- [openai-python v3.13.0 Release](https://github.com/openai/openai-python/releases/tag/v3.13.0)
- [openai-python: Agents API implementation commit](https://github.com/openai/openai-python/commit/1c4284a08294f734d57047585ab82e2e09d3a5bc)
- [PyPI: openai 3.13.0](https://pypi.org/project/openai/3.13.0/)
