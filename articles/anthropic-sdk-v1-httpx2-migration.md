---
title: "Anthropic Python SDK v1のhttpx2移行を安全に検証する"
emoji: "🔌"
type: "tech"
topics: ["anthropic", "python", "httpx", "testing", "ai"]
published: true
---

2026年8月、Anthropic公式Python SDKはv1.0.0でHTTP層を`httpx`から`httpx2`へ移行しました。最新版のv1.4.0では、旧`httpx`のclient、timeout、transportなどを誤って渡したとき、原因と修正先を明示する検査が追加されています。

通常どおり`Anthropic()`を作るだけなら、変更の影響は限定的です。注意が必要なのは、proxy、custom transport、HTTP mock、OpenTelemetryやSentryなど、HTTP層を差し替えたり観測したりするコードです。API呼び出し自体が成功していても、mockやtraceだけがSDKの通信を見失う可能性があります。

この記事では、公式Release、Migration Guide、固定版packageを基に、次を整理します。

- どのコードが移行対象になるか
- v1.4.0のエラー改善を再現する方法
- API keyや外部通信なしでcustom transportを検証する方法
- tracingや既存mockを移すときの判断基準

:::message
2026年9月6日（JST）時点の情報です。SDKは更新頻度が高いため、導入時は最新版をそのまま使わず、Release、Migration Guide、lockfileを照合してください。
:::

## まず公開物を固定する

調査時点で確認した公式配布物は次のとおりです。

| 項目 | 確認結果 |
| --- | --- |
| repository | [`anthropics/anthropic-sdk-python`](https://github.com/anthropics/anthropic-sdk-python) |
| v1でのHTTP層移行 | v1.0.0、2026年8月21日公開（JST） |
| 調査時の最新版 | v1.4.0、2026年9月5日公開（JST） |
| v1.4.0 tag commit | `62de60b27d04f0927a0ccf0f2610597fafcfab6a` |
| Python要件 | 3.10以上 |
| HTTP依存 | `httpx2>=2.0.0,<3` |
| Pydantic依存 | `pydantic>=1.9.0,<3` |
| SDK license | MIT |
| PyPI wheel SHA-256 | `590e85bff75b713a123b03f586d68f02266b5fdc49f70dd75f721ced93a4716c` |

v1.4.0のRelease notesには複数のAPI追加もありますが、本記事は資格情報なしで再現できるHTTP client境界に絞ります。

なお、同日のGitHub Daily TrendingではAgent Skillsやagent harness関連のrepositoryが上位に並んでいました。Trendingは当日の注目度を知る参考値であり、SDKの品質、安全性、互換性を保証するものではありません。本記事の判断はTrendingのstar増加ではなく、固定tag、Migration Guide、PyPI metadata、隔離環境での実行結果に基づきます。

## 影響するコードと、ほぼ影響しないコードを分ける

公式Migration Guideは、`httpx2`をPydanticチームが保守する`httpx`互換forkと説明しています。SDK内部のHTTP処理は`httpx2`へ移りましたが、plain valueだけを渡す一般的な使い方は変更が少ないとされています。

たとえば、次のようなコードはHTTP objectを自分で作っていません。

```python
from anthropic import Anthropic

client = Anthropic(
    timeout=30.0,
    max_retries=3,
)
```

一方、次のいずれかがあれば移行対象です。

- `http_client=`へ`httpx.Client`または`httpx.AsyncClient`を渡す
- `httpx.Timeout`、`httpx.HTTPTransport`、`httpx.Limits`をSDKへ渡す
- `APIStatusError.response`などを`httpx.Response`として型判定する
- `respx`、`pytest-httpx`、`vcrpy`で`httpx`をpatchする
- OpenTelemetryやSentryの`httpx` integrationでSDK通信を観測する
- raw responseの`.text`や`.content`をpropertyとして読む

重要なのは、**アプリケーションが別用途で`httpx`を使うこと自体は問題ではない**点です。SDKへ渡すobjectと、SDKから受け取るHTTP objectの型境界を調べます。

## v1.4.0で何が改善されたか

v1.0.0からSDKへ旧`httpx.Client`を渡すことはできませんでした。v1.4.0の変更は、非互換objectを受け入れることではなく、誤りをHTTP requestの深い場所まで持ち越さず、早い段階で説明することです。

公式commit `94470992f0f00ed331f2faa585cfda2cb8412a08`では、objectのclass hierarchyを調べ、module rootが`httpx`なら`TypeError`を出す検査が追加されました。検査対象には次が含まれます。

- sync / asyncの`http_client`
- client-levelおよびrequest-levelの`timeout`
- default clientへ渡す`transport`

次のコードをPython 3.11の隔離環境で実行しました。

```python
import anthropic
import httpx
import httpx2

print("anthropic", anthropic.__version__)
print("httpx", httpx.__version__)
print("httpx2", httpx2.__version__)

for candidate in (httpx.Client(), httpx2.Client()):
    try:
        client = anthropic.Anthropic(
            api_key="test-key",
            http_client=candidate,
        )
        print(type(candidate).__module__, "accepted")
    except Exception as error:
        print(type(candidate).__module__, type(error).__name__, str(error))
    finally:
        candidate.close()
```

固定版を明示した実行commandは次です。

```bash
uv run --isolated --python 3.11 \
  --with 'anthropic==1.4.0' \
  --with 'httpx==0.28.1' \
  python check_http_client.py
```

実行結果は次のとおりでした。

```text
anthropic 1.4.0
httpx 0.28.1
httpx2 2.12.0
httpx TypeError Invalid `http_client` argument; `httpx.Client` is from the `httpx` package, but this SDK uses `httpx2`. Use `httpx2.Client` instead.
httpx2 accepted
```

比較のため`anthropic==1.3.0`でも同じ検査を行うと、旧clientは同様に拒否されましたが、messageは一般的でした。

```text
httpx TypeError Invalid `http_client` argument; Expected an instance of `httpx2.Client` but got <class 'httpx.Client'>
```

したがって、v1.4.0の改善点を「旧`httpx`互換性の復活」と解釈してはいけません。正しくは、移行漏れをconstruction時に特定しやすくするfail-fastな診断です。

## custom clientは`httpx2`で作る

直接的な移行は、SDKへ渡すHTTP objectのimport元を変えることです。

```python
import httpx2
from anthropic import Anthropic, DefaultHttpxClient

client = Anthropic(
    timeout=httpx2.Timeout(60.0, connect=5.0),
    http_client=DefaultHttpxClient(
        proxy="http://proxy.example",
        transport=httpx2.HTTPTransport(
            local_address="0.0.0.0",
        ),
    ),
)
```

`DefaultHttpxClient`はSDK既定のtimeout、connection limits、redirect設定を維持しながらcustom optionを渡すための公式exportです。

移行では、import文だけでなく次の型も検索します。

```bash
rg 'httpx\.(Client|AsyncClient|Timeout|HTTPTransport|Limits|Response|Request|Headers)' src tests
rg 'DefaultHttpxClient|DefaultAsyncHttpxClient|http_client=' src tests
```

`APIStatusError.response`、`APIConnectionError.request`、raw response内部のHTTP responseも`httpx2`型です。`isinstance(value, httpx.Response)`や型注釈が残っていると、HTTP request本体とは別のerror handling経路で不整合が起きます。

## API keyなしでrequest契約を試す

SDK移行テストのたびに実APIを呼ぶ必要はありません。`httpx2.MockTransport`を使うと、送信先、method、header、response parseをprocess内だけで確認できます。

```python
import anthropic
import httpx2

seen = {}


def handler(request: httpx2.Request) -> httpx2.Response:
    seen["method"] = request.method
    seen["url"] = str(request.url)
    seen["x-api-key"] = request.headers.get("x-api-key")

    return httpx2.Response(
        200,
        json={
            "id": "msg_test",
            "type": "message",
            "role": "assistant",
            "model": "claude-opus-5",
            "content": [{"type": "text", "text": "ok"}],
            "stop_reason": "end_turn",
            "stop_sequence": None,
            "usage": {"input_tokens": 1, "output_tokens": 1},
        },
    )


transport = httpx2.MockTransport(handler)

with httpx2.Client(transport=transport) as http_client:
    client = anthropic.Anthropic(
        api_key="test-key",
        base_url="https://example.invalid",
        http_client=http_client,
    )
    message = client.messages.create(
        model="claude-opus-5",
        max_tokens=1,
        messages=[{"role": "user", "content": "ping"}],
    )

assert message.id == "msg_test"
assert message.content[0].text == "ok"
assert seen == {
    "method": "POST",
    "url": "https://example.invalid/v1/messages",
    "x-api-key": "test-key",
}
```

このtestを`anthropic==1.4.0`、`httpx2==2.12.0`で実行し、assertが通ることを確認しました。これはSDKとtransportの接続、request生成、response parseを確認するcontract testです。実APIの認証、network、model品質、rate limitを検証した結果ではありません。

test用の値でも実在するAPI key形式や本番endpointを使わず、`example.invalid`と明示的なdummy keyを使うと、誤送信を防ぎやすくなります。

## tracingやmockが「静かに消える」問題を検出する

custom clientの型違いはv1.4.0が明示的に拒否します。一方、toolが`httpx`だけをpatchしている場合、SDKのrequestは通常どおり成功し、traceやmockだけが見えなくなる可能性があります。

たとえば、移行前のtestが次の前提を持っている場合です。

```text
application test
    └─ respx / pytest-httpxがhttpxをpatch
           └─ Anthropic SDKもhttpxを使う想定
```

v1ではSDKが`httpx2`を使うため、観測対象が分かれます。

```text
applicationのhttpx通信 ──> 従来のpatch対象
Anthropic SDK通信      ──> httpx2（別のmodule）
```

対策は2通りあります。

### 方法1: SDK境界だけを`httpx2`へ移す

新規コードや、HTTP clientを共有しなくてよいapplicationではこちらを優先します。

- SDK用testに`httpx2.MockTransport`を使う
- SDK用hookを`httpx2`へ付ける
- SDKから受け取るrequest / response型を`httpx2`へ変更する
- applicationの他の`httpx`通信はそのまま残す

影響範囲が局所化され、importの意味が明確です。

### 方法2: process全体でaliasする

公式Migration Guideは、既存toolが`httpx`をpatchするapplication向けに`httpx2.alias_httpx()`を案内しています。

```python
# entry pointの最初に置く
import httpx2

httpx2.alias_httpx()

import httpx

assert httpx.Client is httpx2.Client
```

隔離processで確認すると、`import httpx`は`httpx2`、`import httpcore`は`httpcore2`へ解決されました。

```text
httpx_is_httpx2 True
httpx_module httpx2 httpcore_module httpcore2
```

ただし、これはprocess全体のimport解決を変える操作です。

- **必ず**何かが`httpx`をimportする前に呼ぶ
- library側が利用者に代わって呼ばない
- pytestではtest moduleより早く読み込まれるpluginに置く
- application内の別dependencyが本当に`httpx2`と互換か確認する

すでに`httpx`がimportされた後では`RuntimeError`になるため、「testが失敗したら途中でaliasする」という使い方はできません。

## raw responseとasync処理も同時に棚卸しする

v1移行はHTTP package名だけではありません。公式Migration Guideでは、`.with_raw_response`の返却classも変更されています。

sync clientでは、`.text`と`.content`のproperty accessをmethodへ移します。

```python
# Before
text = response.text
content = response.content

# After
text = response.text()
content = response.read()
```

async clientでは、parseやbody readをawaitします。

```python
response = await client.messages.with_raw_response.create(
    model="claude-opus-5",
    max_tokens=64,
    messages=[{"role": "user", "content": "hello"}],
)

message = await response.parse()
text = await response.text()
content = await response.read()
```

custom transportだけを直してtestが通っても、障害時のraw response記録やasync error pathが未実行なら移行完了とは言えません。

## 安全な移行手順

### 1. dependencyと実行条件を固定する

まず本番環境を変更せず、隔離環境で解決結果を記録します。

```bash
uv run --isolated --python 3.11 \
  --with 'anthropic==1.4.0' \
  python -c 'import anthropic, httpx2; print(anthropic.__version__, httpx2.__version__)'
```

本記事の実行では次になりました。

```text
1.4.0 2.12.0
```

これは`anthropic==1.4.0`が`httpx2`を厳密に2.12.0へ固定しているという意味ではありません。SDKの要件は`>=2.0.0,<3`なので、再現にはlockfileまたはconstraintsが必要です。

### 2. HTTP境界を検索する

次を別々に洗い出します。

1. SDKへ渡すclient、transport、timeout、limits
2. SDKから受け取るrequest、response、headers
3. HTTPをpatchするtest library
4. tracing、APM、error reportingのintegration
5. raw responseとasync responseの処理

単に`requirements.txt`の`httpx`を`httpx2`へ置換すると、application自身が使う`httpx`まで不用意に変える可能性があります。objectがSDK境界を越えるかで分類します。

### 3. construction testを置く

旧objectが残っていれば、v1.4.0ではrequest前に検出できます。

```python
import anthropic
import httpx2

client = anthropic.Anthropic(
    api_key="test-key",
    http_client=httpx2.Client(),
)
client.close()
```

sync / async、proxyあり / なし、custom transportあり / なしを本番構成に合わせて分けます。

### 4. mock transportで送受信を確認する

先ほどの`MockTransport` testで、少なくとも次をassertします。

- methodとpath
- 認証headerの存在（値をlogへ出さない）
- timeoutや追加header
- status error時のresponse型
- response bodyのparse
- sync / async clientのclose

### 5. observabilityを独立して検証する

API成功だけでなく、次の観測結果もtestします。

- spanが1件生成される
- URLにsecretやprompt本文を記録しない
- status codeとrequest IDを取得できる
- retryを重複requestと誤認しない
- mockが意図しない実networkへfallbackしない

trace件数が0でもAPI testがgreenになる構造なら、移行漏れを検出できません。

### 6. canaryしてrollback条件を決める

本番投入前に少量trafficで次を比較します。

| 観点 | 確認内容 |
| --- | --- |
| 接続 | proxy、TLS、DNS、connection pool |
| timeout | connect / read / write / pool timeout |
| retry | 回数、backoff、重複副作用 |
| observability | trace、metric、error event |
| lifecycle | client close、async task、resource leak |
| fallback | rollback後にlockfileとruntimeが一致するか |

version constraintだけを戻し、lockfileに`httpx2`が残ったままにするような半端なrollbackを避けます。

## 導入判定チェックリスト

### Artifact

- [ ] `anthropic`、`httpx2`、Pythonの版を記録した
- [ ] SDKのtagまたはcommitとPyPI artifactを照合した
- [ ] lockfileまたはconstraintsを保存した
- [ ] package sourceとlicenseを確認した

### Code

- [ ] SDKへ渡す`httpx.*` objectを検索した
- [ ] response / requestの型注釈と`isinstance`を確認した
- [ ] raw responseのmethod化とasync `await`を反映した
- [ ] application自身の`httpx`利用とSDK境界を分離した

### Test

- [ ] `httpx2.MockTransport`で実networkなしのcontract testを通した
- [ ] sync / async、成功 / error、retryを確認した
- [ ] mockが未捕捉時に外部通信へfallbackしない
- [ ] traceとmetricが実際に生成されることをassertした

### Operations

- [ ] proxyとTLS設定をcanaryで確認した
- [ ] secretやpromptをtraceへ記録していない
- [ ] connection poolとclient closeを監視した
- [ ] packageとlockfileを戻すrollback手順を試した

## まとめ

Anthropic Python SDK v1の`httpx2`移行で最も危険なのは、明示的な型エラーだけではありません。v1.4.0は旧`httpx` objectを早く分かりやすく拒否しますが、`httpx`だけをpatchするmockやinstrumentationは、SDKのrequestを静かに見失う可能性があります。

安全な順序は次のとおりです。

1. `anthropic`、`httpx2`、Pythonを固定する
2. SDK境界を越えるHTTP objectと型注釈を検索する
3. `httpx2.MockTransport`でrequest / response契約を検証する
4. API成功とは別にtraceとmockの捕捉をassertする
5. proxy、timeout、retry、client lifecycleをcanaryで確認する

単純なimport置換ではなく、「通信」「test」「観測」の3経路を分けて確認すると、SDK upgrade後にAPIだけ動いて監視が消える状態を防げます。

## 参照した一次情報

- [Anthropic Python SDK公式repository](https://github.com/anthropics/anthropic-sdk-python)
- [v1.4.0 Release](https://github.com/anthropics/anthropic-sdk-python/releases/tag/v1.4.0)
- [v1.0.0 Release](https://github.com/anthropics/anthropic-sdk-python/releases/tag/v1.0.0)
- [Migrating to v1](https://github.com/anthropics/anthropic-sdk-python/blob/v1.4.0/MIGRATION.md)
- [v1.4.0のhttpx object診断改善commit](https://github.com/anthropics/anthropic-sdk-python/commit/94470992f0f00ed331f2faa585cfda2cb8412a08)
- [PyPI: anthropic 1.4.0](https://pypi.org/project/anthropic/1.4.0/)
- [httpx2公式repository](https://github.com/pydantic/httpx2)
- [GitHub Trending](https://github.com/trending?since=daily)
