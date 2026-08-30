---
title: "ArchifyでAI生成の構成図を検証可能な成果物にする"
emoji: "🗺️"
type: "tech"
topics: ["ai", "agent", "architecture", "github", "nodejs"]
published: true
---

AIエージェントへ「このシステムの構成図を作って」と頼むと、見栄えのよい図は得られます。しかし、矢印が別のノードを横切る、ラベルが経路を隠す、修正のたびに配置が変わる、失敗した出力で正常版を上書きするといった問題は残ります。

2026年8月30日に公開された[Archify v2.16.0](https://github.com/tt-a1i/archify/releases/tag/v2.16.0)は、AIエージェントが作った型付きJSONを、決定的なHTML/SVGへ変換するツールです。v2.16.0ではWorkflow向けの制約駆動コンパイラが加わり、ノード配置、ラベル、直交経路、コンテナ境界を同じ検査系で扱えるようになりました。

この記事では、公式README、v2.16.0の固定tag、Schema、CLIを基に、次を整理します。

- AIに自由なSVGを書かせる方法と何が違うか
- Workflow schema v2が何を機械検査するか
- `validate`、`deliver`、`visual-check`をどう分けるか
- 固定tagで再現した検証結果
- 自動検査が保証しない範囲

:::message
2026年8月31日7時台（JST）のGitHub Daily Trendingでは、Archifyは34,282 stars、3,730 stars todayでした。数値は取得中にも変動しています。Trending入りは注目度の参考であり、設計の正しさ、品質、安全性、本番適合性の保証ではありません。
:::

## まず公開状況を固定する

GitHub API、Release、固定tagから確認した情報は次のとおりです。

| 項目 | 確認結果 |
| --- | --- |
| repository | `tt-a1i/archify` |
| latest stable release | v2.16.0 |
| 公開日時 | 2026年8月30日20時17分（JST） |
| 固定commit | `c826e6c3a7abad19c0f3cd1ca57207d54b1ad8de` |
| license | MIT |
| runtime | Node.js 18以上 |
| packageの状態 | `private: true`。実体はAgent Skillと同梱CLI |
| 対応する図 | Architecture、Workflow、Sequence、Data Flow、Lifecycle |
| 日本語Viewer UI | 未対応。著者が書く本文は日本語にできるが、固定UIは英語fallback |

リポジトリの`main`は調査時点でtagより先へ進んでいました。再現性が必要なら、READMEの「stable」という表示だけでなく、v2.16.0または40桁のcommit SHAを固定します。

## Archifyは「図を描くAI」だけではない

Archifyの処理は、概念的に2層へ分かれます。

```text
自然言語・コード・既存設計
          ↓
AIエージェント
  └─ 型付きJSON IRを作る
          ↓
Archify CLI
  ├─ JSON Schemaを検証
  ├─ ノードとラベルを計測
  ├─ 経路と衝突を検査
  ├─ HTML / SVGを決定的に生成
  └─ 合格した候補だけ原子的に配置
          ↓
単一のself-contained HTML
```

ここでAIが担当するのは、何をノードにするか、どの関係を記述するか、どのlaneやcolumnへ置くかという**著者判断**です。Archifyは、その記述を機械的にコンパイルし、Schema違反や図形上の不整合を検出します。

自由形式のSVG生成と比べると、JSON IRが残る点が重要です。HTMLだけを直接修正するのではなく、次のような意味を持つデータを差分レビューできます。

```json
{
  "id": "approval-check",
  "from": "router",
  "to": "approval",
  "label": "needs approval?",
  "variant": "security"
}
```

一方、Schemaに合うことは、`router`から`approval`へ本当に処理が流れる証明ではありません。元コードや運用仕様を調べずにAIが関係を作れば、型の正しい誤情報になります。

## v2.16.0のWorkflow schema v2

v2.16.0では、新しいWorkflowに`schema_version: 2`を使うことが推奨されています。schema v1は既存図の固定座標を維持する互換経路、schema v2は`readable-v2`という制約駆動の配置契約です。

最小の考え方は次のようになります。

```json
{
  "schema_version": 2,
  "diagram_type": "workflow",
  "meta": {
    "title": "Release Workflow",
    "quality_profile": "showcase"
  },
  "lanes": [
    { "id": "dev", "label": "Development" },
    { "id": "ops", "label": "Operations" }
  ],
  "nodes": [
    {
      "id": "test",
      "lane": "dev",
      "col": 0,
      "type": "backend",
      "label": "Test"
    },
    {
      "id": "deploy",
      "lane": "ops",
      "col": 1,
      "type": "cloud",
      "label": "Deploy"
    }
  ],
  "edges": [
    {
      "id": "test-deploy",
      "from": "test",
      "to": "deploy",
      "label": "all checks passed"
    }
  ]
}
```

`lane`は責任や工程、`col`は論理的な進行順を表します。schema v2のcolumnは`0..5`です。著者が全座標を先に決めるのではなく、コンパイラが文字幅、ノード幅、lane、経路、ラベルの占有領域を測り、必要なcanvasを計算します。

### layout receiptを先に読む

配置を調べるだけなら、HTMLを作る前に次を実行できます。

```bash
node bin/archify.mjs validate workflow workflow.json --layout-json
```

公式サンプルに実行した結果、次のような機械可読receiptを得ました。

```json
{
  "contract": "readable-v2",
  "viewBox": [1096, 684],
  "requiredViewBox": [1096, 684],
  "columns": [114, 274, 491.2, 651.2, 830, 990],
  "diagnostics": []
}
```

実際のreceiptには、各nodeの`x`、`y`、`width`、`height`、各edgeの全通過点、labelの矩形も含まれます。見た目を勘で直す前に、「どの経路のどのsegmentが問題か」をデータとして読めます。

## showcaseが確認する9項目

最終候補では`showcase` profileを使います。

```bash
node bin/archify.mjs validate workflow workflow.json \
  --quality showcase \
  --json
```

v2.16.0の公式サンプルでは9件すべてが成功し、compositionはerrors 0、warnings 0でした。

| check | 主に確認するもの |
| --- | --- |
| `single_svg` | SVG blockが1つだけか |
| `finite_svg` | `NaN`や無限値がないか |
| `orthogonal_arrows` | 意図しない斜め2点arrowがないか |
| `label_route_clearance` | labelが別のrelationship経路を隠さないか |
| `relationship_crossings` | 無関係な経路の交差がないか |
| `relationship_corridors` | 別の関係が同じ線路を共有して曖昧にならないか |
| `container_border_runs` | edgeがlaneやgroupのborder上を長く走らないか |
| `route_rhythm` | 短すぎるsegmentや窮屈な曲がりがないか |
| `legend_clearance` | legendと図が衝突しないか |

加えて、無関係な不透明nodeをedgeが通過する`clean-flow/edge-through-node`はhard failureです。

重要なのは、検査エラーが単なる「render failed」で終わらない点です。JSON出力には安定したcode、対象、計測値、対応可能な修正が入ります。

```json
{
  "code": "schema/maximum",
  "severity": "error",
  "subject": {
    "path": "/nodes/0/col",
    "identity": "user"
  },
  "evidence": {
    "comparison": "<=",
    "limit": 5
  },
  "supportedFixes": ["use a value <= 5"]
}
```

エージェントへ無制限に再試行させるのではなく、対象fieldと許可された修正を絞れます。

## Atomic Verified Deliveryで正常版を守る

検証と納品は別コマンドです。

```bash
node bin/archify.mjs deliver workflow workflow.json workflow.html \
  --quality showcase \
  --json
```

`deliver`は次の順序で処理します。

1. specificationのbytesを固定する
2. 出力先と同じdirectoryへprivate candidateを作る
3. candidateをrenderする
4. artifactとcompositionを検査する
5. specificationとartifactのSHA-256を記録する
6. すべて成功した場合だけtargetを原子的に置き換える
7. 失敗時はcandidateを削除し、既存targetを維持する

公式サンプルをv2.16.0で実行したreceiptは次でした。

```json
{
  "ok": true,
  "type": "workflow",
  "specification": {
    "sha256": "4d3f6c23de2ca7d306e05d0041a9506833bb5a164a2bddc2d1984cf4da3ee90c",
    "bytes": 5687
  },
  "artifact": {
    "sha256": "325680d835f68cb04089bb5d3fdd1b5671abf1790a15fbcae2167dbe6e756f0b",
    "bytes": 723032
  },
  "validation": {
    "checksPassed": 9,
    "checkCount": 9,
    "compositionProfile": "showcase",
    "errors": 0,
    "warnings": 0
  }
}
```

次に、同じ出力先へ`col: 99`を含む不正なJSONをdeliveryしました。CLIはexit 1で拒否し、delivery前後のHTML SHA-256はどちらも次の値でした。

```text
325680d835f68cb04089bb5d3fdd1b5671abf1790a15fbcae2167dbe6e756f0b
```

つまり今回の検証では、失敗した候補が前回の合格artifactを上書きしないことを確認できました。

## 固定tagで再現する

安全に試すには、global installより先に固定tagを読み取り専用でcloneします。

```bash
workdir="$(mktemp -d)"

git clone --depth 1 --branch v2.16.0 \
  https://github.com/tt-a1i/archify.git \
  "$workdir/archify"

cd "$workdir/archify"
git rev-parse HEAD
```

期待するcommitは次です。

```text
c826e6c3a7abad19c0f3cd1ca57207d54b1ad8de
```

依存をlockfileどおり取得し、公式testを実行します。

```bash
cd archify
npm ci
npm test
```

記事執筆時はNode.js v22.23.1で実行しました。結果は次のとおりです。

```text
added 10 packages
found 0 vulnerabilities
release identity ok: 2.16.0
all checks passed

tests 1010
pass 983
fail 0
skipped 27
```

`npm audit`が0件であることは、未知の脆弱性やSkillの指示内容まで安全だという意味ではありません。また27件のskipを成功へ数えてはいけません。

CLI自体の同梱状態も確認できます。

```bash
node bin/archify.mjs doctor
```

今回の実行では、Node.js、template、5種類のrenderer/schema/example、standalone validator、preview、visual-check、compare runtimeがすべて`[ok]`でした。

## visual-checkは「見た」とは判定しない

`deliver`成功後は、同じHTMLへ次を実行できます。

```bash
node bin/archify.mjs visual-check workflow.html --json
```

このコマンドはChrome/Chromiumを使い、次のdesktop viewportでcontainmentを測ります。

- 1440×900
- 1600×1000
- 1920×1080
- 2048×1320

最小・最大viewportではlight/darkのscreenshotとcontact sheetも作ります。ただしreceiptは常に`visualReview: "pending"`です。

これは正しい区別です。scroll overflowやcapture成功は機械判定できますが、読み順が自然か、強調が適切か、図が過密でないかまでは自動的に証明できません。画像を人が確認していないのに「visual review passed」と報告してはいけません。

また、`deliver`失敗後に既存HTMLへ`visual-check`を実行すると、拒否されたcandidateではなく前回のlast-good artifactを検査してしまいます。先にdelivery errorを直す必要があります。

## 実システムを図にするときの証拠境界

Architecture modeでは、public GitHub repository、full commit SHA、repository-relative source pathを指定する証拠機能があります。ローカルcloneと`--repo-root`を使うと、origin、commit、blob、任意のline rangeを検証してからsource linkをartifactへ入れます。

しかし、source fileへのlinkが正しいことと、図の因果関係が正しいことは別です。

たとえば、同じdirectoryに`queue.ts`と`worker.ts`があるだけで「QueueがWorkerを呼ぶ」と推測してはいけません。実際のimport、message publish/consume、configuration、deployment manifest、runtime entrypointを確認します。

図へ記録する前に、少なくとも次を調べます。

- processとentrypoint
- 同期callと非同期message
- database、cache、object storage
- authenticationとtrust boundary
- retry、timeout、cancel、terminal state
- deployment configurationとexternal dependency
- 対象commitとsource line

Archifyは著者が記述した関係を検査しますが、repository全体を自動解析して真のarchitectureを確定するツールではありません。

## Agent Skillとして導入する前の注意

公式の導入例は次です。

```bash
npx skills add tt-a1i/archify -g
```

ただし、global installを最初の評価手順にしない方が安全です。Agent Skillは`SKILL.md`の指示とNode.js CLIをエージェントへ追加します。固定tagをcloneして内容とtestを確認し、必要ならrepository-local scopeへ入れます。

v2.16.0には、約72時間ごとにstable manifestをGETして任意の更新通知を表示する仕組みがあります。READMEによれば、自動download・自動install・prompt送信は行いません。networkとlocal reminder stateの書込みを止めたい場合は次を設定できます。

```bash
export ARCHIFY_UPDATE_CHECK_DISABLED=1
```

また、unknown brandの画像取得機能を使う場合は外部networkが発生します。v2.16.0はURLとdigestを固定し、private/metadata destinationやremote SVGを拒否する設計ですが、通常の図ではbrandを省略できます。

生成HTMLはself-containedですが、著者がsource code、内部host名、個人情報、秘密情報を書けば、そのままartifactへ残ります。共有前にHTMLとJSONの両方を検査してください。

## 導入を4段階に分ける

### 1. 固定版を読む

v2.16.0をcloneし、`SKILL.md`、対象Schema、1つのexample、license、security policyを読みます。この段階ではglobal skillへ入れません。

### 2. 公式exampleを再現する

`doctor`、`npm test`、`validate --layout-json`、`validate --quality showcase`、`deliver`を実行します。CLIが成功した事実と、自分の設計が正しいことを混同しません。

### 3. 小さな実システムで試す

6〜12個程度の主要componentへ絞り、1本のmain pathから始めます。最初から座標や経路を細かく固定せず、diagnosticが要求した箇所だけ直します。

### 4. CIとreviewへ組み込む

JSON IRとHTMLを成果物として分けます。Pull RequestではJSONのsemantic diffを読み、CIでは`validate`と`deliver`を実行し、必要に応じてscreenshotを人が確認します。

```text
source / design evidence
        ↓ human + agent authorship
     typed JSON IR
        ↓ schema / layout / composition
   verified candidate HTML
        ↓ visual review
      review artifact
        ↓ explicit approval
       shared output
```

## 自動検査が保証しないこと

Archifyの検査結果は有用ですが、次は別の証拠が必要です。

| 主張 | 必要な追加確認 |
| --- | --- |
| 図が実装と一致する | 固定commitのsource、config、runtime trace |
| 全componentを網羅する | scope定義とrepository調査 |
| security boundaryが正しい | IAM、network policy、threat model |
| 変更のblast radiusが分かる | dependency分析、test、運用知識 |
| 見た目が読みやすい | 実browserでの人間によるvisual review |
| production-readyである | 負荷、障害、権限、監視、rollbackの検証 |

Architecture Deltaも、同じstable IDを持つ2つの著者作成snapshotを比較する機能です。追加、削除、変更、移動、rerouteを分類できますが、risk、merge safety、実際のPR影響を推論するものではありません。

## まとめ

Archify v2.16.0の価値は、AIが自動的に正しい構成図を理解することではありません。AIが作った図の仕様を型付きJSONとして残し、機械検査と原子的deliveryを通して、壊れた成果物を渡しにくくすることです。

実務では次を分けて扱います。

1. AIまたは人が事実とトポロジーを記述する
2. JSON Schemaで入力形を検証する
3. layout receiptで配置と経路を調べる
4. showcaseの9項目でartifactを検査する
5. `deliver`で合格版だけを原子的に置き換える
6. browserでvisual reviewする
7. source evidenceとruntime検証で図の内容を裏付ける

固定tagでの検証では、公式test 1,010件中983件成功、27件skip、失敗0でした。公式workflow sampleは9/9 showcaseを通過し、不正入力によるdelivery後も前回HTMLのSHA-256が維持されました。

それでも、検証済みなのは「著者が書いたJSONを、定義された契約どおりにartifactへ変換できたこと」です。設計事実そのものは、固定commit、実装、設定、test、運用記録へ戻って確認する必要があります。

## 参考リンク

- [Archify公式repository](https://github.com/tt-a1i/archify)
- [Archify v2.16.0 Release](https://github.com/tt-a1i/archify/releases/tag/v2.16.0)
- [v2.16.0固定commit](https://github.com/tt-a1i/archify/commit/c826e6c3a7abad19c0f3cd1ca57207d54b1ad8de)
- [Archify README](https://github.com/tt-a1i/archify/blob/v2.16.0/README_EN.md)
- [Archify Skill contract](https://github.com/tt-a1i/archify/blob/v2.16.0/archify/SKILL.md)
- [Workflow layout contract](https://github.com/tt-a1i/archify/blob/v2.16.0/archify/renderers/workflow/README.md)
- [Delivery contract](https://github.com/tt-a1i/archify/blob/v2.16.0/archify/references/delivery-contract.md)
- [Security Policy](https://github.com/tt-a1i/archify/blob/v2.16.0/SECURITY.md)
- [GitHub Trending](https://github.com/trending?since=daily)
