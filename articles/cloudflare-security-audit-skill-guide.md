---
title: "CloudflareのSecurity Audit Skillを安全に評価する"
emoji: "🛡️"
type: "tech"
topics: ["security", "ai", "cloudflare", "github", "agents"]
published: true
---

2026年9月17日（JST）のGitHub Daily Trendingで、Cloudflareの[`security-audit-skill`](https://github.com/cloudflare/security-audit-skill)が確認時点で1,249 stars todayを集めていました。

これはcoding agentへ「脆弱性を探して」と指示するだけのprompt集ではありません。偵察、coverage管理、候補の反証、構造化された判定、独立した再検証を分離し、LLMが出したもっともらしい指摘をそのまま脆弱性として扱わないためのworkflowです。

一方、導入前に理解すべき制約もあります。

- 現行repositoryにはtagとGitHub Releaseがなく、`main`は更新される
- validatorが保証するのは構造と一部の整合性であり、脆弱性の真偽ではない
- targetのbuildやtestを安全に実行するには、単なる一時directoryではなくOSで強制されたsandboxが必要
- 1回の監査で「全件を調べた」とは言えない
- Cloudflareの社内harnessと、公開された単一repository向けskillは同じものではない

この記事では、公式ブログと固定commitのsourceを基に、仕組み、信頼境界、再現可能な検査方法、組織へ導入する前の評価手順を整理します。

:::message
GitHub Trendingとstar数は時間によって変動します。2026年9月17日7時台（JST）のTrending画面は累計6,959 stars、1,249 stars todayを表示し、その後にGitHub APIで取得した累計値は6,999でした。Trending入りは注目度の参考であり、監査品質、安全性、網羅性、本番適合性の保証ではありません。
:::

## まず公開状況を固定する

調査時点の公式配布物は次の状態でした。

| 項目 | 確認結果 |
| --- | --- |
| repository | [`cloudflare/security-audit-skill`](https://github.com/cloudflare/security-audit-skill) |
| default branch | `main` |
| 確認したcommit | `c1c8a8c1471069fb0e188eeaff69b8e8db6564a8` |
| commit日時 | 2026年9月14日19:28:54 UTC |
| commit署名 | GitHub API上で`verified: false`、reasonは`unsigned` |
| tag | なし |
| GitHub Release | なし |
| source license | MIT |
| 実装 | Markdown、JSON Schema、dependency-freeなNode.js validator |
| 対象 | codebase、API、service、CLI、library、daemon |

tagやReleaseがないため、次のinstall commandは調査時点の最新版を取得する手順であって、同じ内容を将来も再現する手順ではありません。

```bash
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit
```

組織で評価するなら、先にsourceを固定します。

```bash
repo='https://github.com/cloudflare/security-audit-skill.git'
commit='c1c8a8c1471069fb0e188eeaff69b8e8db6564a8'
workdir="$(mktemp -d)"

git clone "$repo" "$workdir/security-audit-skill"
git -C "$workdir/security-audit-skill" checkout --detach "$commit"
git -C "$workdir/security-audit-skill" rev-parse HEAD
```

期待する最後の出力は次です。

```text
c1c8a8c1471069fb0e188eeaff69b8e8db6564a8
```

commitがunsignedであることだけを理由に危険と断定はできません。しかし、tagや署名済みReleaseがない状態では、repository URLだけでなく検査済みcommit SHA、取得日時、source hashを導入記録へ残すべきです。

## 公開skillとCloudflare社内harnessを分ける

公式ブログ「Build your own vulnerability harness」は、最初の約450行のskillから、128 repositoriesを扱うfleet-wide systemへ発展させた経緯を説明しています。

ブログに登場する社内systemは大きく2段です。

```text
Vulnerability Discovery Harness (VDH)
  Recon → Hunt → Validate → Gapfill / Dedup / Trace / Feedback → Report
                         ↓
Vulnerability Validation System (VVS)
  Dedup → production contextでJudgment → Fixing → human review
```

一方、公開repositoryは単一codebaseを対象にした出発点です。databaseによる永続化、cross-repository dependency graph、production contextを読む社内MCP、fleet-wide queueまで公開skillに含まれるわけではありません。

ブログは初期skillを7 phaseと説明しますが、固定commitの`SKILL.md`は現行workflowを次の6 phaseとして定義しています。

1. Reconnaissance
2. Coverage-led hunting waves
3. Candidate validation
4. Structured output
5. Independent record verification
6. Target-neutral report

記事や紹介文のphase数ではなく、導入時に固定した`SKILL.md`を実行仕様として扱います。

## 重要なのは「発見」より判定の分離

### 1. 先にtrust boundaryを言語化する

現行skillは、指摘ごとに少なくとも次を要求します。

- lower-trust principalは誰か
- どの入力または操作を受け付けるか
- 本来どのcontrolが守るべきか
- どのboundaryを越えたか
- 誰のresourceへ、どんな具体的結果が出たか

この条件により、次のような一般論をconfirmed vulnerabilityから外します。

- security headerが一つない
- parserへ不正入力を与えると自分のprocessが落ちる
- databaseへのwrite権限を持つ人がdatabaseを書き換えられる
- deployment設定を見ないまま「外部公開されているはず」と推測する

best practice違反と、実際のboundary violationは別です。

### 2. hunterとverifierを別agentにする

候補を見つけたagentには、自分の指摘を確定させません。Phase 3では、その候補を作っていないfresh verifierがsourceを読み直し、可能なら最小の結果を独立に再現します。

判定は3種類です。

| verdict | 意味 | severity |
| --- | --- | --- |
| `confirmed` | source traceとboundedな実行結果が独立に成立した | 付ける |
| `needs_validation` | source上の仮説はあるが、決定的なruntime・deployment事実が観測できない | 付けない |
| `rejected` | source、control、実行結果、影響、前提のいずれかが仮説を反証した | 付けない |

`needs_validation`は「自信の低い脆弱性」ではありません。何が未確認で、誰がどの安全な方法で確認すれば解決するかを残す状態です。

さらにPhase 5では、最終的な`confirmed`だけでなく`needs_validation`もfresh agentが再確認します。誤った保留案件でも、ownerの調査時間を消費するためです。

### 3. coverageをagent数ではなくledgerで管理する

「5 agentで監査した」はcoverageの説明になりません。現行skillは次の組み合わせを安定したunitとして管理します。

```text
entry surface
  × trust boundary
  × subsystem
  × attack class
  × lifecycle（必要な場合）
```

各unitは`planned`、`in_progress`、`covered`、`candidate`、`blocked`、`deferred`などの状態を持ちます。source-only checkとsandbox内のlocal checkも区別します。

重要なのは、1回のpassを完全と呼ばないことです。公式READMEも、Cloudflareのtest runでは単一runが、複数runで見つかった脆弱性全体のおよそ半分しか見つけなかったと説明しています。この値を他のrepositoryへそのまま外挿はできませんが、少なくとも「一度0件だったので安全」という結論は支持しません。

## JSON validatorが保証すること、しないこと

repositoryには2つのdependency-free validatorがあります。

- `validate-findings.cjs`
- `validate-coverage-ledger.cjs`

### findings validatorの役割

`report-schema.json`は3 verdictを別schemaに分けます。たとえば`confirmed`には次が必要です。

- stable fingerprint
- entrypointからsinkまでのtrace
- source evidence
- 成立条件
- attacker perspective、payload、手順、observed result
- remediation
- likelihood、impact、overall severity
- confidence

一方、`needs_validation`にはseverityや実行結果を持たせず、blockerとvalidation planを要求します。`rejected`には反証理由を残します。

validatorはschema適合だけでなく、次も検査します。

- fingerprintの重複とsort順
- severityがdemonstrated impactを超えていないか
- unsafeなsource path
- invalid UTF-8や不正なUnicode scalar
- symlink、FIFO
- 過大なinput、深すぎるnesting、過大なarray
- 制御文字を含むdiagnostic output

固定commitではfindings inputを5 MiB、top-level 1,000件、nesting 64段までに制限し、error出力も100件で打ち切ります。

### coverage validatorの役割

coverage ledgerでは、canonical ID、stateごとの必須evidence、agent ID、artifact ownership、sort順、pathの安全性などを検査します。上限は5 MiB、10,000 units、64段、nested collection 1,000件です。現行`RECONNAISSANCE.md`では、現実的なunitサイズなら5 MiB制限が先に効き、およそ2,000〜5,000 unitsが目安と説明されています。

### validatorが証明しないもの

両方がexit 0でも、次は証明されません。

- traceが本当に実行可能か
- PoCが主張したboundaryを越えたか
- productionで到達可能か
- severityが組織のbusiness impactと一致するか
- 見逃しがないか
- remediationが別のregressionを作らないか

公式ブログも、機械検証はschema adherenceを確認するもので、correctnessそのものではないと区別しています。validatorはLLM出力を次工程へ渡せる形へ制約するgateであり、独立検証の代替ではありません。

## 固定commitでvalidatorを実行する

clone後、repository rootから次を実行します。

```bash
node skills/security-audit/validate-findings.test.cjs
node skills/security-audit/validate-coverage-ledger.test.cjs
```

記事執筆時はNode.js `v22.23.1`で実行し、次の結果になりました。

```text
validate-findings.test.cjs
  tests 34
  pass 34
  fail 0

validate-coverage-ledger.test.cjs
  tests 31
  pass 31
  fail 0
```

最初にrepository root直下の`validate-findings.test.cjs`を呼ぶと、実体は`skills/security-audit/`配下なので`MODULE_NOT_FOUND`になりました。READMEの「Files」表はskill directory内の構成を示しています。automationでは作業directoryと実ファイルpathを固定してください。

自分の監査結果を検証するcommandは次です。

```bash
skill_dir='skills/security-audit'
audit_dir='/監査結果を置いた外部directory'

node "$skill_dir/validate-findings.cjs" \
  "$audit_dir/findings.json"
node "$skill_dir/validate-coverage-ledger.cjs" \
  "$audit_dir/coverage-ledger.json"
```

`audit_dir`はtarget repositoryの外へ置くのが既定です。repository内を選ぶ場合は、明示的に指定し、directory全体がversion controlから除外されていることを確認します。

## target codeは通常のagent shellで実行しない

このskillで最も厳しい前提はexecution safetyです。source inspectionはread-onlyですが、targetが制御するbuild、test、browser、emulator、fuzzer、fixture処理には、次をすべて満たすOS-enforced sandboxを要求します。

1. external networkを遮断する
2. local通信が必要なら隔離したloopback namespaceだけを使う
3. environmentを空にし、安全な値だけallowlistする
4. `HOME`、temp、cacheをscratch内へ置く
5. targetとtoolchainをread-onlyにする
6. target processのwrite先を割り当てた`scratch/`だけにする
7. CPU、memory、process数、file size、disk、wall-clockを制限する
8. dependency取得を許可しない

一つでも強制できない場合はtarget codeを実行せず、決定的な事実が不足する候補を`needs_validation`に残します。

Docker containerを一つ起動しただけでは、自動的にこの条件を満たしません。network、mount、environment、capability、resource limit、nested sandboxの成否を個別に確認する必要があります。

また、target processが作ったartifactは敵対的inputとして扱います。現行skillは、sandbox終了後にtrusted parent processが次を確認して、事前allowlistしたfileだけを昇格する設計です。

- relative pathにabsolute、空、`.`、`..`、symlink componentがない
- no-followかつdirectory descriptor相対で辿る
- leafがregular file、link count 1、size上限内
- 読み取り前後でidentity、type、link count、sizeが変わらない
- destinationもno-followで辿り、排他的に作る
- recursive copy、glob、archive展開をしない
- symlink、FIFO、socket、device、directory、hard linkを昇格しない

これは、監査対象がartifact pathを細工し、host上の秘密や別agentの成果物を読み書きさせる攻撃を防ぐためです。

## 最小評価はguidance modeから始める

現行commitでは、skillをloadしただけではfull auditを許可しません。

- **guidance mode**: security質問、特定箇所のreview、既存findingのtriage
- **full audit mode**: 明示的なcodebase audit、penetration test、包括review、report artifact作成

導入直後から全repositoryへfull auditを回すより、次の順序が安全です。

### Step 1: source-onlyのfocused review

小さなrepositoryまたは狭いsubsystemを選び、buildやtestを実行せず、次を確認します。

- repository-relativeなfile:lineが正しい
- lower-trust principalとprotected resourceが具体的
- preventing controlを読み落としていない
- deployment前提をsource事実として断定していない
- defense-in-depthをvulnerabilityへ格上げしていない

### Step 2:既知の修正済み事例でcalibrationする

組織内で公開可能な、修正済みのsecurity bugを含む過去commitを使います。

```text
入力: 修正前の固定commit
期待: root causeと最小影響をcandidateとして抽出
反証: 修正後のcontrolを適用すると同じclaimがrejectedになる
評価: file/line、条件、影響、severity、fixが人間の記録と一致するか
```

未知の0-dayを何件見つけたかではなく、既知事例でfalse positive、miss、誇張されたimpact、再現不能なPoCを測ります。

### Step 3: sandbox能力を独立にtestする

実際のtargetを入れる前に、無害なfixtureで次を試します。

- external networkへ接続できない
- allowlist外のenvironmentが見えない
- target mountへ書けない
- scratch外へ書けない
- process、memory、disk、time上限で停止する
- symlinkやFIFOをartifactへ昇格できない

sandbox制御をagentの自然言語指示だけで実装しないことが重要です。OS policyとして強制し、失敗時はfail closedにします。

### Step 4: quick profileを部分監査として扱う

`quick`は1回のhunter waveと最終criticを使うbounded passです。速い代わりにpartial coverageであり、「安全」という証明にはなりません。

評価時は次を記録します。

| 観点 | 記録する値 |
| --- | --- |
| source | repository、commit、dirty state |
| scope | path、subsystem、除外範囲 |
| cost | agent invocation、token、wall-clock |
| coverage | covered / candidate / blocked / deferred units |
| candidate quality | confirmed / needs_validation / rejected |
| verifier効果 | hunter候補のうち反証・修正された割合 |
| human effort | triage、再現、修正reviewに要した時間 |
| safety | sandbox violation、network attempt、artifact rejection |

### Step 5: fixは自動mergeしない

公式ブログの社内VVSでも、Fixerはbranchを作れても自動mergeせず、人間がreviewします。評価環境でも次をgateにします。

1. 元のsourceでtarget regression testが失敗する
2. 最小patch後に同じtestが成功する
3. 既存testにregressionがない
4. authorizationやvalidationを別の場所へ移しただけではない
5. 人間がdiff、threat model、deployment影響をreviewする

「AIが脆弱性を見つけた」と「安全なfixをproductionへ入れられる」は別の状態です。

## Cloudflareの数値を自社の性能保証にしない

公式ブログは社内harnessの事例として、20,799 raw candidatesから約12,057件がvalidationを通り、VVSの別入力と合わせた13,841件からdedupやjudgmentを経て7,245 actionable findingsになったと説明しています。

また、standard repositoryの例として約30K LOC、100 initial findings、3〜4時間のrunなども示しています。

これらはCloudflare自身の対象、model、prompt、infrastructure、期間に基づく値です。公開skillを自分のrepositoryへ入れたときのrecall、precision、費用、所要時間を保証しません。公式ブログ自身もfalse-negative rateを主張せず、すべてのreal bugを含むlabeled setがないためrecallは分からないと説明しています。

採用判断では、件数の多さではなく次を見ます。

- independent verifierがどれだけ誤検出を止めたか
- `needs_validation`のblockerがownerにとって具体的か
- 同じroot causeがstable fingerprintでdeduplicateされるか
- rerunでcoverage gapが縮むか
- 人間へ届くunconfirmed findingを減らせたか
- fixのfail→passとregressionを確認できるか

## 導入前チェックリスト

### Artifact

- [ ] repository URLだけでなく40桁commit SHAを固定した
- [ ] tagとReleaseがないことを前提にupdate review手順を決めた
- [ ] MIT licenseと導入先の配布条件を確認した
- [ ] validatorのtestを固定commitで実行した

### Execution safety

- [ ] external networkをOSで遮断した
- [ ] empty allowlisted environmentを強制した
- [ ] targetとtoolchainをread-onlyにした
- [ ] scratch-only writeを強制した
- [ ] CPU、memory、process、file、disk、timeを制限した
- [ ] artifact promotionをno-followかつsize-boundで実装した
- [ ] 制御が欠ける場合は実行せず`needs_validation`へ落とす

### Evidence

- [ ] findingごとにattacker、boundary、resource、resultを説明できる
- [ ] hunterとverifierが別である
- [ ] `needs_validation`へseverityを付けていない
- [ ] validator成功をcorrectness証明と呼んでいない
- [ ] live endpointやproduction identityを監査trafficでprobeしていない

### Operations

- [ ] quick/scoped runをpartial coverageと報告する
- [ ] coverage ledgerを各更新後にvalidateする
- [ ] rerunで同じroot causeを重複ticket化しない
- [ ] target-controlled artifactをそのまま保存しない
- [ ] AI生成patchを自動mergeしない
- [ ] source、delivered、deployedの状態を分ける

## まとめ

Cloudflareの`security-audit-skill`から学べる最大のポイントは、強いmodelへ長いpromptを渡すことではありません。LLMの不安定さを前提に、次の境界を明示することです。

1. reconnaissanceとhuntingを分ける
2. candidateを見つけたagentに自己採点させない
3. `confirmed`、`needs_validation`、`rejected`を混ぜない
4. JSON validatorで形式とstate invariantを制約する
5. validator成功と脆弱性の正しさを混同しない
6. target codeをOS-enforced sandboxの外で実行しない
7. 1回のrunを完全なcoverageと呼ばない
8. patchの適用にはfail→pass、regression test、人間reviewを残す

まず固定commitのvalidator testを通し、source-onlyの狭いreviewと既知の修正済みbugでcalibrationします。sandboxとartifact promotionを実装できない段階では、動的検証を無理に実行せず、source-groundedな候補を`needs_validation`として管理する方が安全です。

## 参照した一次情報

- [Cloudflare: security-audit-skill repository](https://github.com/cloudflare/security-audit-skill)
- [固定commit `c1c8a8c`](https://github.com/cloudflare/security-audit-skill/commit/c1c8a8c1471069fb0e188eeaff69b8e8db6564a8)
- [Cloudflare Blog: Build your own vulnerability harness](https://blog.cloudflare.com/build-your-own-vulnerability-harness/)
- [Security Audit SKILL.md](https://github.com/cloudflare/security-audit-skill/blob/c1c8a8c1471069fb0e188eeaff69b8e8db6564a8/skills/security-audit/SKILL.md)
- [report-schema.json](https://github.com/cloudflare/security-audit-skill/blob/c1c8a8c1471069fb0e188eeaff69b8e8db6564a8/skills/security-audit/report-schema.json)
- [validate-findings.cjs](https://github.com/cloudflare/security-audit-skill/blob/c1c8a8c1471069fb0e188eeaff69b8e8db6564a8/skills/security-audit/validate-findings.cjs)
- [validate-coverage-ledger.cjs](https://github.com/cloudflare/security-audit-skill/blob/c1c8a8c1471069fb0e188eeaff69b8e8db6564a8/skills/security-audit/validate-coverage-ledger.cjs)
- [GitHub Trending](https://github.com/trending?since=daily)
