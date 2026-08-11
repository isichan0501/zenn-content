---
title: "DeepTutor v1.5.11から学ぶ学習エージェントの壊れ方"
emoji: "🎓"
type: "tech"
topics: ["ai", "agent", "education", "github", "security"]
published: true
---

2026年8月12日のGitHub Trendingでは、学習エージェント[DeepTutor](https://github.com/HKUDS/DeepTutor)が確認時点で829 stars todayを集めていました。

DeepTutorは、chat、quiz、deep research、問題演習、knowledge base、memory、subagentを1つの学習workspaceへまとめるApache-2.0のopen source projectです。

今回注目したのは機能数ではなく、最新版v1.5.11の修正内容です。

- tool callと同じroundにある説明文が消える
- token上限で切れた回答を「完了」と誤認する
- local indexingがevent loopを止める
- quizの正解をmodelが書き換え得る
- model生成codeをhost上で実行する設定が既定で有効

これらは教育専用の問題ではありません。streaming、tool use、memory、RAG、code executionを組み合わせる全エージェントに共通する運用課題です。

:::message
GitHub Trendingとstar数は時間によって変動します。本記事の数値は2026年8月12日7時6分（JST）のスナップショットです。Trending入りは注目度の参考であり、学習効果、品質、安全性、本番適合性の保証ではありません。
:::

## 固定版の公開状況

GitHub API、Release、固定tagのソースから次を確認しました。

| 項目 | 確認結果 |
| --- | --- |
| repository | `HKUDS/DeepTutor` |
| 累計stars | 34,654 |
| 当日増加 | 829 stars today |
| 最新安定版 | v1.5.11 |
| v1.5.11公開日時 | 2026年8月10日1時48分（JST） |
| 固定commit | `456f9c24226e008f1ff07a7e3455d7b4d39f6221` |
| license | Apache-2.0 |
| backend | Python 3.11〜3.13 |
| frontend | Next.js 16、source開発ではNode.js 22 LTS推奨 |
| upstream CI | 固定commitのTests workflowがsuccess |

リポジトリは2025年12月末に公開され、2026年8月までに1.5系へ進んでいます。直近だけでも数日おきにreleaseがあります。活発な開発は利点ですが、再現性が必要な導入では`latest`ではなくtagと40桁SHAを記録すべき更新速度です。

## DeepTutorを「賢いchat」だけとして見ない

READMEはDeepTutorをagent-native learning workspaceと説明しています。主なsurfaceは次のとおりです。

- Chat、Quiz、Research、Visualize、Solve、Mastery Path
- LlamaIndex、PageIndex、GraphRAG、LightRAG、Obsidianによるknowledge base
- L1 trace、L2 summary、L3 synthesisのmemory
- Claude Code、Codexなどのsubagent / Partner
- MCP server、CLI app、community skill
- 文書生成のためのcode execution

構造を単純化すると、次のruntimeです。

```text
学習者の質問・教材・過去memory
  ↓
agent loop
  ├─ LLM生成
  ├─ tool call
  ├─ RAG / web / paper search
  ├─ quiz・grading
  ├─ subagent
  └─ code execution
  ↓
streaming回答・artifact・memory更新
```

機能が増えるほど、最終回答の正しさだけでは品質を測れません。途中のevent、tool argument、権限、永続化、停止判定を別々に検証する必要があります。

## v1.5.11が示す5つの故障モード

### 1. Tool callの前後にある説明文が消える

DeepSeek系の一部endpointは、native function callingではなく、DSMLというmarkupをcontent channelへ出す場合があります。

```text
説明文
<DSML invoke name="tool">...</DSML invoke>
続きの説明文
```

従来のfilterは、tool callを含むroundのcontentを大きく別channelへ流し、call前後のproseまで最終回答から失わせる場合がありました。

v1.5.11の`DSMLStreamFilter`は、次の方針へ変わっています。

1. 完全なenvelopeと`invoke` blockだけを抑止する
2. tool callの前、間、後にある通常文を保持する
3. chunk境界をまたぐtagをbufferする
4. 未完の`invoke`は`flush()`時に原文を戻す
5. 512文字を超える未完tag候補は通常文として解放する

Streamingでは、完成した文字列を最後にregex処理するだけでは不十分です。`<`、tag名、parameter、終了tagが別chunkに分割され得ます。

安全なfilterには、少なくとも次の不変条件が必要です。

- 通常文を欠落させない
- 完全なtool markupを利用者へ露出しない
- 不完全なprovider出力を黙って捨てない
- 任意のchunk分割で結果が変わらない
- malformed inputで無限にbufferしない

### 2. token切れを「モデルが終了を選んだ」と扱う

LLM APIには、自然に完了した応答と、長さ上限で切れた応答があります。どちらも次のroundへ進まなければ最終回答に見えますが、意味は異なります。

v1.5.11では、`length`によるfinishを意図的終了とみなさず、見えているprefixを保持したまま継続を要求します。同時に、loopが無制限にならないよう、概念的に次の3段階へbudgetを分けています。

```text
exploration
  ↓ 上限到達
bounded settlement window
  ↓ まだ完了しない
1回のtool-less hard finish
```

重要なのは、継続を許すことと、必ず止めることを両立させる点です。

- 早く止めすぎると回答が欠ける
- 無制限に続けるとcostとlatencyが増える
- toolを突然外すと進行中のinteractionが壊れる
- 完了判定をmodel任せにすると同じcallを繰り返す

教育用途では、解説の結論やquizの選択肢が欠けても文章として自然に見える場合があります。UIに`partial`、`continued`、`hard_finished`を別状態で表示する方が安全です。

### 3. Async関数の中に同期処理が隠れる

LightRAG indexingでは、graph mergeやJSON serializationのような同期処理がasync method内で動き、他のAPIやLLM処理が使うevent loopをstallさせる問題がありました。

v1.5.11はlocal indexingをworker thread専用のevent loopへ移し、LLM、vision、embedding callだけを元のrequest loopへbridgeします。

`async def`と書かれていることは、処理がnon-blockingである証明ではありません。次を含む処理は計測が必要です。

- 大きなJSONのserialize
- graph merge
- PDF parse
- vector index構築
- CPU-bound embedding前処理
- 同期filesystem I/O

runtime評価では平均response timeだけでなく、indexing中に別利用者のchat latencyがどう変わるかを測ります。

```text
p50 latency
p95 latency
p99 latency
最大event-loop lag
indexing throughput
peak RSS
cancel後に残るworker数
```

### 4. Quizの正解をmodelへ再記述させない

Mastery機能では、保存済みのquestionから公開用contractをserver側で作り、期待するanswerはserver側へ残すよう修正されています。`ask_user`が表示するquestionも、modelが言い換えたdisplay dataではなくpersisted questionを基準にします。

これは小さく見えて重要な境界です。

```text
authoritative question store
  ├─ prompt / choices → 学習者へ表示
  └─ expected answer → server-side gradingだけで使用
```

modelへ「問題と正解をJSONで再出力して」と頼むと、選択肢の順序、label、否定、正解そのものが変わる可能性があります。採点対象はmodel outputではなく、versionを持つserver-side recordにします。

### 5. Code executionは明示的なtrust decision

DeepTutorのoffice skillは、modelがPython codeを書き、DOCX、PDF、PPTX、XLSXなどを生成します。READMEによれば、local / single-container構成ではrestricted subprocess sandboxが既定で有効です。

設定の既定値はsource上でも次のとおりです。

```json
{
  "sandbox_allow_subprocess": true
}
```

local installでは、model生成codeがhost上のsubprocessとして動くことを意味します。READMEもreal trust decisionだと明記しています。

不要なら、起動前に無効化します。

```bash
export DEEPTUTOR_SANDBOX_ALLOW_SUBPROCESS=0
```

またはSettingsで`sandbox_allow_subprocess`をfalseにします。この場合、code executionを必要とするoffice skillは動かなくなります。

より強い構成として、docker-composeではleast-privileged runner sidecarを利用できます。ただしcontainer化だけで安全性が自動的に完成するわけではありません。mount、network、capability、secret、resource limitを確認します。

## 固定tagを読み取り専用で検査する

導入前にsourceを実行せず、固定版を確認できます。

```bash
workdir="$(mktemp -d)"
repo="$workdir/DeepTutor"

git clone --depth 1 --branch v1.5.11 \
  https://github.com/HKUDS/DeepTutor.git \
  "$repo"

git -C "$repo" rev-parse HEAD
python3 -m compileall -q "$repo/deeptutor"
```

期待するcommitは次です。

```text
456f9c24226e008f1ff07a7e3455d7b4d39f6221
```

記事執筆時はこのtagをcloneし、`deeptutor/`配下の648個のPython fileへ`compileall`を実行し、構文検査が成功しました。

さらに、DSML parserをI/OやLLMなしで直接読み込み、次の3条件を確認しました。

1. 完全なtool callを構造化し、前後の通常文を保持する
2. 3文字ごとのstream chunkでも同じ通常文を返す
3. 未完の`invoke`をflush時に原文のまま返す

3条件はすべて成功しました。

:::message
`compileall`とparserの直接検査は、Full app、RAG、provider接続、frontend、sandbox、学習効果を検証するものではありません。localのpytestは最小環境で依存`pydantic-settings`がなくcollectionできなかったため、成功したとは報告しません。代わりに、同じ固定SHAに対する上流の`Tests` workflowがsuccessであることをGitHub上で確認しました。
:::

## 初回起動は権限を狭くする

DeepTutorにはPyPI、source、single Docker、CLI-onlyの経路があります。評価時はhost環境へ直接入れず、固定tagと隔離環境を使います。

### 最初は生成codeを無効にする

```bash
export DEEPTUTOR_SANDBOX_ALLOW_SUBPROCESS=0
```

まずchat、quiz、knowledge retrievalなど、code executionを必要としない機能を評価します。

### Dockerではloopbackだけへ公開する

公式READMEのsingle-container例はfrontendをloopbackへbindします。

```bash
docker run --rm --name deeptutor \
  -p 127.0.0.1:3782:3782 \
  -v deeptutor-data:/app/data \
  ghcr.io/hkuds/deeptutor@sha256:ee1cd852d5514c2ec71e463d3a119102677d17d029cc2e55539602ee5a822e67
```

記事執筆時にGHCR Registry APIを照会し、`1.5.11` tagがこのmulti-platform manifest digestを指すことを確認しました。tagは後から別manifestを指し得るため、本番では取得時のdigestを記録し、承認済みdigestへ固定します。`latest`を再現単位にしません。

### 設定directoryをversion管理しない

READMEではmodel profile、API key、auth設定、integration設定などがworkspaceの`data/user/settings/`へ置かれます。

| file | 主な内容 |
| --- | --- |
| `model_catalog.json` | provider profile、API key、model |
| `system.json` | port、CORS、sandbox、upload limit |
| `auth.json` | auth、password hash、cookie設定 |
| `integrations.json` | PocketBaseなどのintegration |

これらを教材repositoryや共有Gitへcommitしません。backupする場合も暗号化、access control、retentionを別途設計します。

## 学習効果とruntime品質を分けて測る

DeepTutorの機能が正常に動いても、学習効果が上がったとは限りません。評価を3層へ分けます。

### 1. Runtime

- answerが欠けずに完了したか
- tool callがschemaどおりか
- 無限loopや重複callがないか
- indexing中も他requestへ応答できるか
- cancel、再起動、障害復旧が正しいか

### 2. Content

- 教材・sourceにgroundされているか
- 解法と答えが一貫するか
- 不確実性を表示するか
- citationが元資料の主張を支えるか
- 同じ問題の言い換えで答えが変わらないか

### 3. Learning outcome

- pre-testとpost-testで改善したか
- 後日のretentionが上がったか
- hintなしでtransfer問題を解けるか
- modelへの依存が増えていないか
- 誤答を自信を持って覚えていないか

「利用時間が長い」「会話数が多い」はengagementであり、理解や定着の直接指標ではありません。

## Memoryは正しさより来歴を確認する

DeepTutorはL1 trace、L2 summary、L3 synthesisというinspectable memoryを掲げています。層を分ける設計は有用ですが、要約を重ねるほど元発言から離れる可能性があります。

学習memoryには、次のfieldを明示的に持たせます。

| field | 記録する内容 |
| --- | --- |
| `learner_id` | scopeを限定した仮名ID |
| `claim` | 理解度についてsystemが推論した内容 |
| `evidence_ids` | 実在するquiz attemptや会話traceのID |
| `source_layer` | L1 / L2 / L3のどの層で作られたか |
| `created_at` | 実際の作成時刻 |
| `expires_at` | 再評価または失効させる時刻 |
| `review_status` | 未検証、人間確認済み、訂正済みなどの状態 |

値がないfieldを推測で補わず、不要な個人情報も入れません。

確認すべき点は次です。

- どのquiz、会話、教材から推論したmemoryか
- 学習者が閲覧、訂正、削除できるか
- 古い理解度が失効するか
- 別user、別course、Partnerへ漏れないか
- summaryから原traceへ戻れるか
- 正解を見た後の回答を「自力で理解」と誤認しないか

## Multi-userでは認証だけでなく所有権を試す

v1.5.10では、各userのCodex sign-in情報が管理者の共有catalogへ公開されないよう、owner単位のcatalogへ保存する修正が入っています。これはmulti-user systemで重要な教訓です。

resource isolation testでは、2つのtest accountを用意し、次を確認します。

1. user Aのmodel profileをuser Bが一覧できない
2. user Aのknowledge baseをuser Bが参照できない
3. user AのmemoryをPartnerやadmin以外が取得できない
4. output artifact URLを推測しても取得できない
5. WebSocket reconnect後もowner contextが混ざらない
6. export、backup、logにもowner IDとaccess controlが残る

UIで見えないだけでは不十分です。API、WebSocket、artifact path、background workerの各経路を直接testします。

## 本番導入前のチェックリスト

- [ ] v1.5.11または承認済みSHAを固定した
- [ ] Release、license、upstream CIを確認した
- [ ] Python、Node.js、container imageのversionを固定した
- [ ] `latest`ではなくtag / digestを使った
- [ ] host subprocess executionの要否を決めた
- [ ] 不要なら`DEEPTUTOR_SANDBOX_ALLOW_SUBPROCESS=0`にした
- [ ] secretを設定file、log、教材、Gitへ入れていない
- [ ] tool call前後のproseとpartial responseをtestした
- [ ] indexing中のp95 / p99 latencyを測った
- [ ] quizのauthoritative answerをserver側へ保持した
- [ ] memoryのsource、scope、expiry、deleteを確認した
- [ ] 2 accountでcross-user accessをnegative testした
- [ ] 学習効果をengagementとは別に測った
- [ ] provider障害、rate limit、model変更時のfallbackを試した

## まとめ

DeepTutor v1.5.11から学べるのは、多機能な学習エージェントほど、modelの回答品質だけでは評価できないことです。

- streaming filterは任意のchunk境界で通常文を失わない
- token切れと意図的な完了を別状態にする
- settlementを許しつつhard limitで停止する
- CPU-bound処理をrequest event loopから分離する
- quizの問題と正解をmodel出力ではなくserver recordで管理する
- code executionを明示的な権限として無効化可能にする
- memoryとcredentialをowner単位で隔離する

DeepTutorはApache-2.0で、固定tag、詳細なrelease note、上流testが揃った活発なprojectです。一方、local構成ではmodel生成codeのsubprocess実行が既定で有効であり、更新頻度も高いため、無条件に`latest`を本番へ入れる対象ではありません。

まず固定版をread-onlyで監査し、code executionを無効にした隔離環境で、runtime、content、learning outcomeを別々に測るのが安全です。

## 参考リンク

- [DeepTutor公式リポジトリ](https://github.com/HKUDS/DeepTutor)
- [DeepTutor v1.5.11 Release](https://github.com/HKUDS/DeepTutor/releases/tag/v1.5.11)
- [v1.5.11固定commit](https://github.com/HKUDS/DeepTutor/commit/456f9c24226e008f1ff07a7e3455d7b4d39f6221)
- [v1.5.11 Tests workflow](https://github.com/HKUDS/DeepTutor/actions/runs/31324753146)
- [DeepTutor paper（arXiv:2604.26962）](https://arxiv.org/abs/2604.26962)
- [DeepTutor Documentation](https://deeptutor.info/)
- [GitHub Trending](https://github.com/trending?since=daily)
