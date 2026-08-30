---
title: "OpenMAIC v1の耐久Agent Runtimeを安全に試す"
emoji: "🏫"
type: "tech"
topics: ["ai", "agent", "education", "security", "github"]
published: true
---

2026年8月27日、Open Multi-Agent Interactive Classroom（OpenMAIC）v1.0.0が公開されました。1つの指示や教材から、スライド、クイズ、対話、シミュレーション、PBL（Project-Based Learning）を含むコースを作るオープンソースの学習環境です。

v1.0.0の重要な変更は、生成画面が増えたことだけではありません。コース作成Agentの実行状態をPostgreSQLへ保存し、workerが止まっても再開できる**Durable Agent Runtime**が追加されました。利用者は実行中のAgentへ追加指示を送り、処理を取り消し、過去のeventを再生できます。

ただし、公式の環境変数テンプレートは、このAgent Runtimeを`experimental`、既定OFFと明記しています。また、同梱される永続化APIの開発用tokenは、公開環境のユーザー認証には使えません。

この記事では、公式README、v1.0.0 Release、固定tagの実装、公開論文ページを基に、次を整理します。

- 耐久セッションが通常のchat履歴と何が違うか
- lease、heartbeat、cancel、timeoutが解く故障
- URL取得と教材uploadで守るべき境界
- そのまま公開してはいけない開発用認証
- 固定tagを検証し、安全にPoCを始める手順

:::message
GitHub Trendingとstar数は時間によって変動します。本記事では2026年8月30日22時34分（JST）のDaily Trendingを確認し、OpenMAICは23,093 stars、907 stars todayでした。Trending入りは注目度の参考であり、学習効果、品質、安全性、本番適合性の保証ではありません。
:::

## まず公開状況を固定する

確認した一次情報は次のとおりです。

| 項目 | 確認結果 |
| --- | --- |
| repository | `THU-MAIC/OpenMAIC` |
| release | v1.0.0 |
| release公開日時 | 2026年8月27日22時13分（JST） |
| 固定commit | `aa2bfb3c1d406c47100c6744d90e788abdf1f6d5` |
| root license | MIT |
| bundle内の例外 | `mathml2omml`はLGPL-3.0-or-later、同梱`pptxgenjs`はMIT |
| runtime | Node.js 20.9.0以上、公式package managerはpnpm 10.28.0 |
| main app | Next.js 16.1.2、React 19.2.3 |
| Agent Runtime | experimental、既定OFF、PostgreSQL必須 |
| 公式demo | 調査時にHTTP 200 |

rootがMITでも、repository全体の再配布では同梱packageごとの条件を確認します。「GitHubのLicense欄がMITだから全fileが同じ条件」とは限りません。

また、v1.0.0という番号を「すべての機能がproduction-ready」と読み替えないことが重要です。main appのrelease番号と、個別機能の成熟度は別です。Agent Runtimeは`.env.example`でexperimentalとされ、明示的に有効化しない限り`/api/agent/sessions*`と`/api/agent/owner-events`は404を返す設計です。

## OpenMAICが作るもの

従来のclassic modeは、入力からoutlineを作り、各項目をsceneへ展開する二段構成です。sceneにはスライドだけでなく、クイズ、interactive HTML、PBLなどがあります。生成後はPowerPoint、interactive HTML、classroom ZIPへ出力できます。

v1.0.0のPro workbenchでは、この一括生成を会話型の作業へ広げます。

```text
利用者の指示・upload教材・Web資料
                ↓
       Course-building Agent
       ├─ curriculumを計画
       ├─ course / folderを作成
       ├─ sceneを生成・編集・並べ替え
       ├─ quiz / PBL / interactiveを作成
       ├─ 画像・動画・音声を生成
       └─ PPTXをimport
                ↓
     PostgreSQL-backed session
       ├─ message
       ├─ tool call / result
       ├─ status
       ├─ event transcript
       └─ lease / heartbeat / attempt
                ↓
      編集可能なclassroom artifact
```

公式READMEでは20個のbuilt-in skillが案内されています。ここでのskillは、curriculum planning、Feynman、spiral curriculum、slide craft、PPTX importなど、Agentがcourse toolをどう使うかをまとめた手順です。

重要なのは、Agentが最終的なJSON blobを自由に書き換えるのではなく、course作成・scene patch・material参照などの**検証されたtool境界**を通る点です。ただし、toolが構造を検証することと、教材内容が正確で教育的に有効であることは別です。

## Durable Agent Runtimeが解く4つの故障

長い教材生成では、LLMが1回応答すれば終わるとは限りません。資料の抽出、Web検索、複数sceneの生成、画像や音声の作成をまたぐため、数分以上の処理になります。

### 1. workerが落ちてもsessionを失わない

通常のin-memory job queueでは、process restart時に「どこまで終わったか」が消える場合があります。OpenMAIC v1はsession、message、event、materialなどをserver側へ保存し、background runnerが未完了sessionをclaimします。

実行状態を会話本文だけでなく、機械判定できるfieldへ分離するのが要点です。

```text
queued
  ↓ claim + lease
running
  ├─ heartbeat更新
  ├─ tool eventを追記
  ├─ follow-upを取り込む
  ├─ cancelを検出
  └─ worker停止 → lease失効
                      ↓
                 別workerが再claim
```

単にchat logを保存するだけでは、途中のtoolが完了したか、retryしてよいか、別workerが同じ仕事を実行中かを判断できません。

### 2. leaseとheartbeatで二重実行を抑える

環境変数テンプレートに示される既定値は次です。

```env
OPENMAIC_AGENT_RUNTIME_SCAN_INTERVAL_MS=1000
OPENMAIC_AGENT_RUNTIME_HEARTBEAT_MS=2000
OPENMAIC_AGENT_RUNTIME_LEASE_TTL_MS=10000
OPENMAIC_AGENT_RUNTIME_MAX_CONCURRENT=2
OPENMAIC_AGENT_RUNTIME_MAX_ATTEMPTS=5
```

runnerはsessionを無期限に所有するのではなく、短いleaseを取り、heartbeatで延長します。workerが停止してheartbeatが途切れれば、lease失効後に別workerが引き継げます。

ただし、leaseだけでは外部副作用のexactly-once実行を保証できません。たとえば外部の画像生成APIが成功した直後、結果保存前にworkerが落ちれば、再開後に同じrequestを送る可能性があります。

外部providerを追加するときは、次を別途確認します。

- provider側にidempotency keyがあるか
- request IDとresult IDを永続化しているか
- retry時に既存resultをread-backできるか
- upload、課金、公開などの副作用を重複させないか
- attempt上限後に人間へ戻せるか

### 3. cancelを単なるUI表示で終わらせない

長時間taskのcancelには競合があります。

1. 利用者がcancelを要求する
2. workerはすでにtool callを実行中
3. tool resultがcancel後に返る
4. 古いworkerがsessionを再びrunningへ戻す

v1.0.0のRelease noteは、cancel requestのatomicな消費、durable tool writeのfencing、cancel済みsessionを復活させない修正を挙げています。また、実行中TTSのabortとprovider requestのtimeoutも追加されています。

これは「cancel buttonがある」より重要です。確認すべき不変条件は次です。

```text
cancelled session
  ├─ 新しいtool callを開始しない
  ├─ 遅れて返ったresultでrunningへ戻らない
  ├─ owner eventへ取消を記録する
  └─ 再開する場合は別の明示操作にする
```

### 4. hung toolでsession全体を止めない

全tool callには既定10分のglobal timeoutがあります。

```env
OPENMAIC_AGENT_TOOL_TIMEOUT_MS=600000
```

解決もrejectもしないtool callはabortされ、error resultとしてAgentへ返ります。session自体は即座にfailedにせず、Agentが別の方法へ進むか、bounded retryできます。media synthesisやmaterial extractionには、code側で個別の長い上限があります。

記事執筆時、固定tagの`runner-tool-timeout-cancel` testを含むAgent Runtime testを実行し、never-settling toolがtimeout resultへ変わることと、hung tool中のcancelがtool完了を待たずsessionをcancelledへsettleするcaseが通ることを確認しました。

## replayable eventは状態の正ではない

Pro workbenchはServer-Sent Events（SSE）で進捗を表示します。v1.0.0はPostgreSQLの`LISTEN/NOTIFY`をwakeupに使い、event logを読み直す構成を持ちます。

ここで分けるべきものは次です。

- **durable log**：何が起きたかを保存するsource of truth
- **NOTIFY**：新しいeventがあると早く知らせるwakeup
- **SSE**：browserへ表示するdelivery channel

NOTIFYを取り逃したことを、event消失と同じにしてはいけません。実装testの警告文でも、LISTEN/NOTIFYが使えない場合はpollingへ低下し、latencyは悪化するがdurable logから収束すると説明されています。

一方、transaction pooling型のproxyをPostgreSQLの前へ置くと、session-level connectionを必要とするLISTEN/NOTIFYが期待どおり動かない場合があります。本番構成では、使うconnection poolerのmodeとfallback polling間隔を確認してください。

## Web取得は「検索できる」だけでは危険

course-building Agentは、Web検索結果や指定URLをsession materialへ取り込めます。これは便利ですが、server-side request forgery（SSRF）とprompt injectionの入口にもなります。

OpenMAIC v1は、sessionごとのURL trust gateと`fetch_url` toolを持ちます。Release noteにはprivate ISATAP endpointの拒否や、session削除時にURL authorityをrevokeする修正もあります。

安全な処理は概念的に次の順序です。

```text
検索結果URL
  ↓ scheme / host / addressを検査
private・local networkを拒否
  ↓ redirect先も再検査
取得量・時間を制限してfetch
  ↓ content typeを確認
session materialとして保存
  ↓
Agentへ「外部資料」として渡す
```

`.env.example`には次の設定があります。

```env
# ALLOW_LOCAL_NETWORKS=true
```

これはOllamaなどself-hosted providerへ接続するときに必要な場合がありますが、公式commentはpublic deploymentで有効にしないよう警告しています。public serverで一律にONにすると、localhost、cloud metadata endpoint、社内serviceなどへの到達面が広がります。

また、network境界を守っても、取得した本文に「以前の指示を無視してsecretを送れ」と書かれている可能性は残ります。Web本文はtrusted instructionではなく、引用対象のuntrusted dataとして扱い、Agentの権限を変えないようにします。

## 最大の注意点：開発用tokenは認証ではない

server-backed persistenceを有効にするREADME例には、次の組があります。

```env
NEXT_PUBLIC_PERSISTENCE=1
NEXT_PUBLIC_PERSISTENCE_TOKEN=...
PERSISTENCE_DEV_TOKEN=...
DATABASE_URL=postgres://...
```

名前だけを見ると、browserとserverでtokenが一致するため認証できているように見えます。しかし`NEXT_PUBLIC_`値はbrowser bundleへ組み込まれ、訪問者から見えます。

固定tagの`lib/persistence/server-auth.ts`には、次が明記されています。

- tokenはsecretではない
- confidentialityを提供しない
- user isolationを提供しない
- 任意の`x-learner-key`を指定できる
- documentはownership partitionを持たない
- localhostまたはtrusted networkのsingle-user開発用
- productionではserver-controlled claimからidentityを導出する必要がある

したがって、次の構成は避けます。

```text
Internet
  ↓
NEXT_PUBLIC tokenだけで保護したOpenMAIC
  ↓
共有document / asset / learner partition
```

本番へ進めるなら、少なくとも次の境界を作ります。

1. IdPまたはserver-side sessionで利用者を認証する
2. browserが送ったowner IDやlearner keyを信じない
3. server側のclaimからowner / learner partitionを導出する
4. stage、material、asset、session、skillの全routeでownershipを強制する
5. object storageの署名URLにも同じauthorizationを適用する
6. 管理者、教員、学習者、閲覧者のroleを分ける
7. cross-user negative testをAPI単位で実行する
8. access logへtoken、教材本文、provider keyを残さない

`ACCESS_CODE`もsite-levelの入口を狭める補助にはなりますが、複数userの所有権分離やresource単位のauthorizationの代わりにはなりません。

## upload教材は4つの境界を通す

PDF、Word、PowerPoint、spreadsheet、画像、音声、動画を扱うsystemでは、LLM以前にfile処理のriskがあります。

### 1. 受付

- MIME typeと拡張子を照合する
- stream中の実sizeを上限と比較する
- ownerごとのquotaを予約する
- filenameをpathとして直接使わない

固定tagのAgent Runtime testには、unsupported MIMEの415、body超過の413、owner quota超過の429、保存失敗時のreservation cleanupが含まれています。

### 2. 保存

- object keyをserver側で生成する
- ownerとmaterial IDをmetadataへ保存する
- DB確定前後のorphan byteを回収する
- deleteとgarbage collectionにgrace periodを設ける

READMEではasset collectorが既定15分間隔、未参照byteのgrace periodが既定1時間と説明されています。値を変えるときは、storage costだけでなく、誤削除から戻せる時間も考えます。

### 3. 抽出

- parserを隔離する
- CPU、memory、時間を制限する
- cloud parserへ送る場合はdata residencyを確認する
- fallbackで外部cloudへ勝手に送らない

MinerUの設定では、self-hostedが未設定でもcloudへ暗黙fallbackしない設計です。`ALLOW_MINERU_CLOUD_FALLBACK=true`を明示した場合だけ外部送信を許します。教材に未公開研究、学生情報、著作物が含まれる場合、この区別は重要です。

### 4. Agentへの入力

抽出成功は内容の正しさを保証しません。OCR誤り、表の崩れ、page欠落、音声認識誤りを検査し、source pageやtimestampへ戻れる形を残します。教材中の命令文をsystem instructionとして扱わないことも必要です。

## 固定tagを読み取り専用で検証する

まずv1.0.0を固定します。

```bash
workdir="$(mktemp -d)"
git clone --depth 1 --branch v1.0.0 \
  https://github.com/THU-MAIC/OpenMAIC.git \
  "$workdir/OpenMAIC"
cd "$workdir/OpenMAIC"
git rev-parse HEAD
```

期待するSHAは次です。

```text
aa2bfb3c1d406c47100c6744d90e788abdf1f6d5
```

Node.jsとpackage managerを確認し、lockfileを固定して依存を取得します。

```bash
node --version
corepack pnpm --version
corepack pnpm install --frozen-lockfile
```

公式`package.json`はNode.js 20.9.0以上、pnpm 10.28.0を指定しています。記事執筆時はNode.js 22.23.1、pnpm 10.28.0でinstallを完了しました。

Agent Runtimeの非PostgreSQL専用testを実行します。

```bash
pnpm exec vitest run tests/agent-runtime \
  --exclude '**/*.pg.test.ts'
```

実行結果は次でした。

```text
Test Files  70 passed | 1 skipped (71)
Tests       904 passed | 3 skipped (907)
```

skipしたのはPostgreSQL実instanceを必要とするtestです。代わりに、同じcommitに対するupstream CIの`Storage Contract (PostgreSQL 16)`がsuccessであることをGitHub上で確認しました。`Lint, Typecheck & Unit Tests`、`E2E Tests`、`Render Service`、CodeQL Analyzeもsuccessでした。

production buildも実行します。

```bash
pnpm build
```

Next.jsのoptimized production build、TypeScript、52 static pagesの生成は成功しました。`instrumentation.ts`の`process.once`についてEdge Runtime警告が2件出ましたが、buildはexit 0です。警告を成功と偽装せず、Edgeへ配置するrouteがある場合はruntime指定とimport chainを追加確認します。

:::message
この検証は固定tagのinstall、package build、Agent Runtimeのunit/integration相当test、upstream CI状態を確認したものです。実provider API、実教材、学習成果、multi-user production認証、負荷試験、障害注入を検証したものではありません。
:::

## 最小PoCは機能を段階的に有効化する

最初からpublic multi-user serviceにせず、loopbackまたは隔離networkで1人のoperatorが評価します。

### 段階1：classic modeだけを確認

```env
DEFAULT_MODEL=openai:YOUR_MODEL
OPENAI_API_KEY=...
```

この段階ではAgent Runtimeとserver persistenceを有効にしません。1つの短い教材でoutline、scene、exportを確認します。

### 段階2：PostgreSQLとAgent Runtimeを限定的に有効化

```env
NEXT_PUBLIC_PRO_WORKBENCH_ENABLED=true
OPENMAIC_AGENT_RUNTIME_ENABLED=true
DATABASE_URL=postgres://openmaic:***@postgres:5432/openmaic
MODEL_ROUTES={"maic-agent-driver":{"model":"openai:YOUR_MODEL","api":"openai-completions"}}
```

`maic-agent-driver`はprovider prefix付きmodelと、`openai-completions`または`openai-responses` dialectを明示する必要があります。意図しないvendor fallbackはなく、routeがなければsession開始時に失敗します。

PoCでは次を試します。

1. session開始後にprocessを再起動し、再開できるか
2. tool実行中にcancelし、sessionが復活しないか
3. follow-upを連続送信して重複配信されないか
4. provider timeout時にbounded retryで停止するか
5. private URLとredirect先が拒否されるか
6. quota超過、DB停止、object store失敗を区別できるか
7. event stream切断後にdurable logから追いつくか

### 段階3：教材uploadを追加

個人情報や未公開資料を入れず、公開済みの短いPDFから始めます。抽出textと元pageを比較し、表、数式、画像、音声timestampの欠落を記録します。

### 段階4：public化の前に認証moduleを置き換える

開発tokenのままInternetへ公開しません。2つのtest accountを作り、次のnegative testを行います。

- user Aのsession IDをuser Bが取得できない
- user Aのstage IDをuser Bがpatchできない
- user Aのmaterial IDやasset URLをuser Bが読めない
- user Aのskillをuser Bが変更・削除できない
- owner headerやlearner keyを書き換えても権限が変わらない
- SSE reconnect後もowner contextが混ざらない
- export、backup、logにも同じaccess controlが残る

UIで一覧に出ないだけでは不十分です。HTTP routeとobject storageを直接testします。

## 教材生成品質はruntime成功と分けて測る

sessionが再開でき、testが通り、course fileが生成されても、学習に役立つとは限りません。評価を3層へ分けます。

### Runtime

- sessionが重複実行されない
- cancel、retry、resumeが正しい
- tool callとresultが欠けない
- uploadとstorage quotaが守られる
- timeout、費用、latencyが上限内

### Content

- sourceにない事実を作っていない
- 引用が元pageを支える
- quizの正解と解説が一致する
- 数式、図、codeが正しい
- 対象年齢・前提知識・言語が一貫する
- generated interactiveが意図した状態遷移を持つ

### Learning outcome

- pre-test / post-testで改善したか
- 後日のretentionが上がったか
- 別問題へtransferできるか
- hintなしでも解けるか
- 流暢なAI教師に誤った自信を持たないか

OpenMAICの関連論文はLLM-driven agentsによるonline learningの構想とmulti-agent classroomを扱いますが、self-hostした自分のcourseが同じ効果を持つとは限りません。model、prompt、教材、学習者、言語、評価条件を固定し、自分の利用場面で測ります。

## 導入前チェックリスト

- [ ] v1.0.0または承認済みcommit SHAを固定した
- [ ] root licenseとbundled packageのlicenseを分けて確認した
- [ ] Agent Runtimeがexperimental・既定OFFであることを理解した
- [ ] PostgreSQL backup、migration、connection modeを設計した
- [ ] lease、heartbeat、attempt、timeoutの上限を決めた
- [ ] external providerの重複課金・重複生成を検査した
- [ ] cancel後にsessionやtool writeが復活しないことを試した
- [ ] private network、redirect、prompt injectionをnegative testした
- [ ] `ALLOW_LOCAL_NETWORKS`をpublic deploymentで有効にしていない
- [ ] cloud parserへの教材送信を明示opt-inにした
- [ ] 開発用public tokenをproduction認証として使っていない
- [ ] server claimからowner / learner identityを導出した
- [ ] 2 accountでsession、stage、material、asset、skillを隔離した
- [ ] runtime、content、learning outcomeを別々に評価した
- [ ] provider key、教材本文、個人情報をlogへ残していない

## まとめ

OpenMAIC v1.0.0は、multi-agent classroomの見た目だけでなく、長いcourse-building taskを運用するための状態層を追加しました。

- sessionをPostgreSQLへ保存する
- leaseとheartbeatでworker停止から再開する
- follow-up、cancel、event replayをdurableに扱う
- tool callをtimeoutし、session全体のhangを避ける
- URL trust gateとowner-scoped resourceを持つ
- model routeを明示し、暗黙fallbackを避ける

一方、v1.0.0というrelease番号でもAgent Runtimeはexperimentalです。特に同梱の`NEXT_PUBLIC_PERSISTENCE_TOKEN`方式は開発用で、user isolationを提供しません。

最初の導入では、固定tagをbuild・testし、loopback上のsingle-user PoCでrestart、cancel、timeout、URL拒否を確認してください。public化はその後です。教材を生成できること、長期taskを正しく運用できること、複数userのdataを隔離できること、実際に学習効果があることを、別々の証拠で確認する必要があります。

## 参考リンク

- [OpenMAIC公式repository](https://github.com/THU-MAIC/OpenMAIC)
- [OpenMAIC v1.0.0 Release](https://github.com/THU-MAIC/OpenMAIC/releases/tag/v1.0.0)
- [v1.0.0固定commit](https://github.com/THU-MAIC/OpenMAIC/commit/aa2bfb3c1d406c47100c6744d90e788abdf1f6d5)
- [OpenMAIC Changelog](https://github.com/THU-MAIC/OpenMAIC/blob/v1.0.0/CHANGELOG.md)
- [OpenMAIC環境変数テンプレート](https://github.com/THU-MAIC/OpenMAIC/blob/v1.0.0/.env.example)
- [開発用persistence認証の実装](https://github.com/THU-MAIC/OpenMAIC/blob/v1.0.0/lib/persistence/server-auth.ts)
- [From MOOC to MAIC（JCST）](https://jcst.ict.ac.cn/en/article/doi/10.1007/s11390-025-6000-0)
- [GitHub Trending](https://github.com/trending?since=daily)
