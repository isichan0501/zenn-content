---
title: "OpenCodeReviewをCIへ入れる前に確認する5つの境界"
emoji: "🔍"
type: "tech"
topics: ["ai", "codereview", "githubactions", "security", "go"]
published: true
---

2026年9月18日（JST）のGitHub Daily Trendingで、Alibabaの[`open-code-review`](https://github.com/alibaba/open-code-review)が確認時点で3,290 stars todayを集めていました。

OpenCodeReviewは、変更ファイルの選択やルール適用を決定的な処理に寄せ、判断が必要な箇所をLLMエージェントへ任せるコードレビューCLIです。単にdiffを長いpromptへ貼るのではなく、関連ファイルのgrouping、repository内の検索、行位置の補正、結果の構造化までを一つのpipelineとして扱います。

一方、CIへ追加すると次の境界が生まれます。

1. 未公開コードを外部のLLM providerへ送る境界
2. GitHub tokenでPRへcommentする境界
3. `latest`と固定versionの境界
4. command成功とレビューcoverageの境界
5. 公式benchmarkと自分のrepositoryでの品質の境界

この記事では、最新Release `v1.12.5`の固定source、公式documentation、GitHub Action定義を基に、安全に評価する手順を整理します。

:::message
GitHub Trendingとstar数は時間によって変動します。2026年9月18日7時台（JST）のTrending画面は累計34,505 stars、3,290 stars todayを表示し、その後にGitHub APIで取得した累計値は34,569でした。Trending入りは注目度の参考であり、レビュー品質、安全性、本番適合性の保証ではありません。
:::

## まず公開状況を固定する

調査時点の公式配布物は次の状態でした。

| 項目 | 確認結果 |
| --- | --- |
| repository | [`alibaba/open-code-review`](https://github.com/alibaba/open-code-review) |
| latest Release | `v1.12.5` |
| Release公開日時 | 2026年9月17日22時33分（JST） |
| Release commit | `189be5b024d3309dd10fdc8cd8ee31b2530c210b` |
| npm package | `@alibaba-group/open-code-review@1.12.5` |
| source license | Apache-2.0 |
| 主な実装 | Go 1.25.5 |
| 対応LLM | OpenAI互換、Anthropic、Gemini、Bedrockなど |
| maturity | GitHub上でpre-releaseではない正式Release |

Git tag `v1.12.5`はannotated tagで、tag objectの署名はGitHub API上で`verified: false`、reasonは`unknown_key`でした。tagが指すcommit自体はGitHubの署名検証で`verified: true`です。

署名状態だけで安全・危険を決めることはできません。評価記録にはtag名だけでなく、解決した40桁commit SHA、npm integrity、取得日時を残します。

```bash
repo='alibaba/open-code-review'
tag='v1.12.5'

# annotated tagが指すcommitを確認する
tag_object="$(gh api "repos/$repo/git/ref/tags/$tag" --jq '.object.sha')"
gh api "repos/$repo/git/tags/$tag_object" --jq '.object.sha'

npm view @alibaba-group/open-code-review@1.12.5 \
  version dist.tarball dist.integrity dist.unpackedSize
```

今回確認したcommitとnpm integrityは次のとおりです。

```text
189be5b024d3309dd10fdc8cd8ee31b2530c210b
sha512-+8gdJ90ipWrBmdF1rQHt4EcCGn3cvJIos/ZSRJkdFWUHWlV8Bc2rU/gBjWnlQVmauFwkOfHJyTAc7BstwlfPuA==
```

Releaseには各OS向けbinaryと`sha256sum.txt`があります。例えばApple Silicon向けbinaryのSHA-256は、調査時点で次でした。

```text
505bfce5c3d609efb358dfcb911b5ed6726bfcf5ee7f1ed0a57acce13bec4751  opencodereview-darwin-arm64
```

installerを直接実行する前に、Release assetをdownloadし、同じReleaseのchecksumと照合する方が変更内容を追跡しやすくなります。

## v1.12.5で変わった「予算上限」の意味

`v1.12.5`のRelease notesには、`--max-tokens-budget`を実行中のgroup内でも強制する修正が含まれます。これは費用管理に有用ですが、coverageの解釈には注意が必要です。

公式GitHub Actionの定義では、token予算を超えると次の動作になります。

- 予算超過中のgroupには、findingを提出する最後のroundを1回与える
- 新しいgroupをdispatchしない
- 予算超過・未処理のfileを`failed(budget)`として報告する
- partial resultを公開する
- review processはexit 0にできる

つまり、workflowがgreenでも、対象fileをすべて十分にレビューしたとは限りません。CIの成功状態とレビューcoverageは別に監視する必要があります。

最低限、runごとに次を保存します。

```text
selected files
reviewed files
skipped files
failed(budget) files
group count
LLM requests
total tokens
comments generated
```

「commentが0件だった」場合も、安全だったのか、差分が対象外だったのか、context上限や予算でskipされたのかを分けます。

## 境界1：どのコードがLLM providerへ送られるか

公式READMEによると、OpenCodeReviewはGit diffだけでなく、agent toolを使って次を参照できます。

- file全体
- 他の変更file
- repository内の検索結果
- review ruleや背景情報

したがって「変更行だけが外部へ送られる」と仮定してはいけません。導入前にproviderのdata handling、保存期間、学習利用、地域、契約上の機密情報の扱いを確認します。

### 送信前の除外はrepository単位で検証する

対象外にすべきものの例です。

- 秘密鍵、token、credentialを含むfixture
- 顧客データや本番log
- 未公開の脆弱性情報
- 契約上、第三者providerへ送信できないsource
- 生成物、vendor directory、大きなlock file

除外ruleを書いただけで終わらず、意図的なcanary文字列を含むfixture repositoryで、どのfileが選択されるかを確認します。provider側の監査機能が利用できる場合も、秘密そのものをcanaryに使わず、無害な識別子を使います。

### local modelは通信境界を減らすが、品質保証にはならない

公式設定はOllamaなどのOpenAI互換endpointも扱えます。

```bash
ocr config set provider                         ollama
ocr config set custom_providers.ollama.url      http://127.0.0.1:11434/v1
ocr config set custom_providers.ollama.protocol openai
ocr config set custom_providers.ollama.model    '<tool-call対応model>'
```

local endpointを使えばsourceの外部送信を減らせますが、modelがnative tool callingに対応している必要があります。また、誤検出、見逃し、長いdiffでのcontext不足は別に評価します。

## 境界2：API keyとsession transcriptを分ける

公式configurationにはAPI keyを設定fileへ保存する方法もありますが、CIではGitHub Secretsまたはsecret managerからrun単位で渡します。

local評価では`api_key_cmd`を使い、keyを設定fileへ直接書かない選択肢があります。

```bash
ocr config set providers.anthropic.api_key_cmd \
  'secret-manager-cli read ai-review-key'
```

`api_key_cmd`はshell commandとして実行されるため、設定file自体がtrusted inputです。公式documentationでは設定fileを`0600`で書くと説明していますが、所有者と書込み権限も確認します。

API keyをlogへ出さないことと、コード内容がdiskへ残らないことは別です。公式documentationでは次の保存を説明しています。

- review sessionのJSONL transcriptはlocal disk上の`~/.opencodereview/`に残る
- OpenTelemetryは既定で無効
- telemetryを有効にしてもpromptとresponse本文は送らない
- `OCR_RAW_LOGGING=1`を有効にすると、LLM request / response bodyをlocal diskへそのまま保存する

CIではraw loggingを無効のままにし、session artifactをuploadする場合は公開範囲、保存期間、削除方法を決めます。GitHub Actionの`upload_artifacts`は既定で`true`なので、組織の要件に合わせて明示設定すべきです。

## 境界3：Actionの権限と外部PRを分ける

公式ActionはPRへinline commentとsummaryを投稿できます。必要以上の権限を与えない初期設定は次です。

```yaml
permissions:
  contents: read
  pull-requests: write
```

`resolve_outdated: true`は、古くなったthreadを実際にresolveする変更操作です。公式Action定義では追加の`contents: write`が必要とされています。最初は次の順で進めます。

1. `resolve_outdated: false`で何も変更しない
2. `resolve_outdated: report`で候補だけをlogへ出す
3. 人間が対象を確認する
4. 必要な場合だけ権限と`true`を有効にする

外部forkのPRでは、LLM tokenを使えるeventと、untrustedなPR contentを実行するeventを混同しないことが重要です。OpenCodeReviewのActionはtrustedなbaseをcheckoutし、PR headをGit objectとして取得する設計ですが、workflow全体に別のscriptやbuild stepを追加すると、その安全境界は変わります。

特に`pull_request_target`でPR側のscriptを実行しない、secretを渡したjobでPR由来のcommandを評価しない、というGitHub Actions一般の原則は維持します。

## 境界4：versionを二重に固定する

GitHub Actionをcommit SHAへ固定しても、Action inputの`ocr_version`は既定で`latest`です。Action sourceと、実際にinstallされるnpm packageを別々に固定します。

```yaml
- name: Review with OpenCodeReview
  uses: alibaba/open-code-review@189be5b024d3309dd10fdc8cd8ee31b2530c210b
  with:
    ocr_version: "1.12.5"
    llm_url: ${{ secrets.OCR_LLM_URL }}
    llm_auth_token: ${{ secrets.OCR_LLM_TOKEN }}
    llm_model: ${{ vars.OCR_LLM_MODEL }}
    llm_use_anthropic: "false"
    upload_artifacts: "false"
    sticky_summary: "true"
    incremental: "false"
    resolve_outdated: "false"
    max_tokens_budget: "50000"
```

これは最小の設定例であり、すべてのrepositoryへそのまま適用する完成形ではありません。provider protocol、review対象、予算、artifact保持、forkの扱いは組織ごとに決めます。

更新時は、Action SHAとnpm versionの両方を変え、Release notes、dependency差分、権限差分をreviewします。`latest`を使うと、同じworkflow fileでも翌日の挙動が変わります。

## 境界5：公式benchmarkを自社品質へ読み替えない

公式READMEはAACR-Benchを次の規模で説明しています。

- 人気OSS 50 repositories
- 実際の200 Pull Requests
- 10 programming languages
- 80人超のsenior engineerによるcross-validation
- 1,505 annotated ground-truth issues

READMEは、同じ基盤modelを使ったgeneral-purpose agentとの比較で、OpenCodeReviewがprecisionとF1、token消費、時間で優れ、recallは低いと説明しています。

ここで重要なのは、precisionを優先するtrade-offを公式自身が明記していることです。誤検出が少ないことと、重大な問題を見逃さないことは同じではありません。また、公式benchmarkの結果を今回の調査環境で再実行したわけではありません。

採用評価では自分のrepositoryから、公開可能な過去事例を作ります。

| test case | 期待する確認 |
| --- | --- |
| 修正済みbugの修正前commit | root causeを検出できるか |
| 同じbugの修正後commit | 古い指摘を繰り返さないか |
| 大きなgenerated diff | 正しく除外またはskip理由を出すか |
| 関連fileをまたぐ変更 | repository contextを使えるか |
| 無害なstyle変更 | 不要なblocking commentを増やさないか |
| secretに似たfixture | provider送信前の除外が機能するか |
| token budget超過 | partial coverageを明示できるか |

評価単位はcomment数ではなく、次に分けます。

```text
precision = 正しい指摘 / 全指摘
recall    = 検出した既知問題 / 用意した既知問題
location  = comment位置が実際のdiff範囲と一致した割合
noise     = 人間が却下・無視したcomment数
latency   = PR作成からreview完了まで
cost      = provider請求とtoken数
coverage  = reviewed / selected / skipped / failed files
```

重大度別にも集計します。style指摘を大量に当てても、authorizationやdata lossに関わる既知bugを見逃すならblocking gateには使えません。

## 固定版のsource testを実行する

今回、`v1.12.5`が指すcommitをdepth 1で取得し、Go 1.26.5 / macOS arm64で次を実行しました。

```bash
git clone --depth 1 --branch v1.12.5 \
  https://github.com/alibaba/open-code-review.git
cd open-code-review
git rev-parse HEAD
go test ./...
```

確認したcommitは次です。

```text
189be5b024d3309dd10fdc8cd8ee31b2530c210b
```

`go test ./...`はexit 0で、`cmd/opencodereview`と`internal/`配下の全packageが成功しました。GitHub上でも同commitに対するCIとCodeQL workflowは成功しています。

ただし、この結果が保証するのは固定sourceのtestが今回の環境で通ったことです。次は保証しません。

- 利用するLLM modelでレビュー品質が十分である
- 自社の言語・framework・規約を理解できる
- providerへ送るdataが契約上許可されている
- GitHub comment権限が最小である
- 既知の脆弱性をすべて検出できる

source testとレビュー品質評価は別工程です。

## 段階導入の手順

### Step 1：local・read-onlyで固定版を評価する

いきなりPRへcommentせず、固定版を使ってJSONへ出力します。

```bash
ocr review \
  --from main \
  --to feature-branch \
  --format json \
  --output result.json \
  --effort low \
  --max-tokens-budget 50000
```

対象file、skip理由、finding、token使用量を人間が確認します。結果fileにはsource断片が含まれ得るため、公開artifactへ置きません。

### Step 2：人間のreviewと並走させる

2〜4週間はblockingにせず、通常のcode reviewと同じPRへ結果を出します。人間が先に付けた指摘をgold standardと決めつけず、修正後に「正しい・誤検出・重複・重要度過大・見逃し」を分類します。

### Step 3：comment routingでnoiseを制御する

公式Actionには、一定以下のseverityやcategoryをinline commentではなくsummaryへ送る機能があります。findingを捨てずに、開発者のinterruptを減らせます。

最初はstyleやdocumentationをsummaryへ送り、bugとsecurityだけinlineにするなど、repositoryの運用へ合わせます。未知のcategoryやseverityはroutingされずinlineに残るfail-open設計なので、想定外の値も監視します。

### Step 4：coverageとcostへalertを付ける

OpenTelemetryは既定で無効です。有効にすると、review時間、file数、comment数、LLM request、token、tool callなどを出せます。公式documentationではprompt / response本文をtelemetryへ付与しないと説明されています。

ただし、group keyやrepository directoryなどのmetadataも組織によっては機密です。collectorへ送る前にattribute一覧を確認します。

alert例です。

- `failed(budget)`が1件以上
- selected fileに対してreviewed fileが急減
- token / changed lineが過去平均との差を超える
- review latencyがPRのSLOを超える
- comment 0件かつskip理由が空
- provider error後もworkflowが成功扱い

### Step 5：blocking gateは限定したルールから始める

AI finding全体を直ちにmerge blockerへしません。まず決定的に検証できるrule、既知bugでprecisionとrecallを測れたcategory、再現手順のある高重大度findingに限定します。

OpenCodeReviewのRoadmapも、自動修正を人間のreviewなしで適用する機能は計画外としています。suggested fixが出ても、自動mergeせず、testのfail→pass、既存test、権限・data flowの差分を人が確認します。

## 導入前チェックリスト

### Artifact

- [ ] Actionを40桁commit SHAへ固定した
- [ ] `ocr_version`も具体的なnpm versionへ固定した
- [ ] npm integrityまたはRelease checksumを保存した
- [ ] source licenseがApache-2.0であることを確認した
- [ ] 更新時にActionとpackageの両方をreviewする

### Data

- [ ] diff以外のfile内容もproviderへ送られ得ると理解した
- [ ] providerの保存、学習利用、地域、契約条件を確認した
- [ ] secret、顧客data、未公開脆弱性の除外をfixtureで試した
- [ ] local sessionとartifactの保存期間を決めた
- [ ] `OCR_RAW_LOGGING`をCIで有効にしていない

### GitHub

- [ ] 既定権限を`contents: read`、`pull-requests: write`から始めた
- [ ] `resolve_outdated`を`false`または`report`で評価した
- [ ] fork PRとsecretを同じuntrusted stepへ渡していない
- [ ] Action以外のscriptがPR側codeを実行しないことを確認した
- [ ] bot commentの重複・再実行方針を決めた

### Quality and operations

- [ ] 自社の修正済みbugでprecisionとrecallを測った
- [ ] severity別・言語別・diff size別に評価した
- [ ] token budget超過を成功扱いの完全レビューと誤認しない
- [ ] selected / reviewed / skipped / failedを保存した
- [ ] AI findingを最初からmerge blockerにしていない
- [ ] suggested fixを人間reviewなしでmergeしない

## まとめ

OpenCodeReviewの価値は、LLMへ自由にレビューさせるだけでなく、file選択、grouping、rule matching、行位置、結果投稿を決定的な工程で囲むことです。特に`v1.12.5`のtoken budget修正は、費用上限を実行中のgroupにも適用する実務的な改善です。

一方、CI導入では次を分離する必要があります。

1. Trendingでの注目度と、自社repositoryでの品質
2. workflowのexit 0と、全fileのレビュー完了
3. Action sourceの固定と、npm package versionの固定
4. API keyの保護と、source code・transcriptの保護
5. AIのsuggestionと、人間が承認した修正

まず固定版をlocalでJSON出力し、過去の修正済みbugを使ってprecision、recall、coverage、costを測ります。その後、非blockingなPR comment、summaryへのnoise routing、限定的なgateの順に広げると、便利さと統制を同時に検証できます。

## 参照した一次情報

- [Alibaba OpenCodeReview repository](https://github.com/alibaba/open-code-review)
- [OpenCodeReview v1.12.5 Release](https://github.com/alibaba/open-code-review/releases/tag/v1.12.5)
- [v1.12.5 fixed commit](https://github.com/alibaba/open-code-review/commit/189be5b024d3309dd10fdc8cd8ee31b2530c210b)
- [OpenCodeReview Configuration](https://github.com/alibaba/open-code-review/blob/v1.12.5/pages/src/content/docs/en/configuration.md)
- [OpenCodeReview Telemetry](https://github.com/alibaba/open-code-review/blob/v1.12.5/pages/src/content/docs/en/telemetry.md)
- [OpenCodeReview CI/CD Integration](https://github.com/alibaba/open-code-review/blob/v1.12.5/pages/src/content/docs/en/integrations/ci.md)
- [OpenCodeReview Security Assurance Case](https://github.com/alibaba/open-code-review/blob/v1.12.5/ASSURANCE_CASE.md)
- [OpenCodeReview Roadmap](https://github.com/alibaba/open-code-review/blob/v1.12.5/ROADMAP.md)
- [AACR-Bench dataset](https://huggingface.co/datasets/Alibaba-Aone/aacr-bench)
- [GitHub Trending](https://github.com/trending?since=daily)
