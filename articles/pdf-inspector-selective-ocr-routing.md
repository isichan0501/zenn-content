---
title: "pdf-inspectorでPDFのOCR要否をページ単位に振り分ける"
emoji: "📄"
type: "tech"
topics: ["ai", "pdf", "ocr", "python", "rust"]
published: true
---

RAGやAIエージェントへPDFを渡す処理で、すべてのページを最初からOCRへ送っていないでしょうか。PDFには、文字を直接抽出できるもの、画像しかないもの、正常な文字層とスキャンページが混在するものがあります。一律の処理は、不要な遅延と費用を増やすだけでなく、元から存在した文字情報をOCRの誤認識で置き換える原因にもなります。

2026年8月18日（JST）に公開された[Firecrawlのpdf-inspector v1.15.0](https://github.com/firecrawl/pdf-inspector/releases/tag/v1.15.0)は、PDFを分類し、native extractionで読めないページだけをOCRへ振り分けるselective OCRを追加しました。

この記事では、公式README、v1.15.0の固定tag、Python API、同梱fixtureを基に、次を整理します。

- PDF全体ではなくページ単位でOCR要否を決める理由
- `detect_pdf`、`process_pdf`、`process_pdf_with_ocr`の役割分担
- 固定版で再現したtext-based / image-basedの判定結果
- OCR modelとnative runtimeを運用時に分離する方法
- 作者公開のbenchmarkを採用判断へ使う際の注意点

:::message
2026年9月1日7時台（JST）のGitHub Daily Trendingでは、`firecrawl/pdf-inspector`は17,323 stars、199 stars todayと表示されていました。数値は変動します。Trending入りは注目度の参考であり、抽出品質、安全性、本番適合性の保証ではありません。
:::

## まず公開状況を固定する

調査時点のGitHub API、Release、package manifestから確認した内容です。

| 項目 | 確認結果 |
| --- | --- |
| repository | [`firecrawl/pdf-inspector`](https://github.com/firecrawl/pdf-inspector) |
| 調査対象release | v1.15.0 |
| 公開日時 | 2026年8月18日3時7分（JST） |
| 固定commit | `06a9bab6b3309309503f2db17851389cee094a62` |
| license | MIT |
| 実装 | Rust。Python、Node.js、WebAssembly bindingあり |
| Python要件 | CPython 3.8以上 |
| v1.15.0の位置付け | selective OCRを追加したstable release |

Releaseには、Rust crate、Python、Node.js、WebAssemblyの各packageが同じsource checkpointからbuildされたと明記されています。再現時は単に最新版を入れるのではなく、versionを固定します。

```bash
uv run --isolated \
  --with 'pdf-inspector==1.15.0' \
  python inspect_pdf.py document.pdf
```

main branchはrelease後も更新されます。挙動を比較するときは、package versionに加えて40桁のcommit SHAも記録すると、後からsourceへ戻れます。

## OCRを二択ではなくroutingとして扱う

pdf-inspectorが扱う分類は次の4種類です。

- `text_based`：native textを抽出できる
- `scanned`：スキャン画像を中心とする
- `image_based`：有用な文字層がなく、画像として処理すべき
- `mixed`：native extractionとOCRが必要なページが混在する

処理を概念化すると、次のようになります。

```text
PDF
 └─ detect_pdf
      ├─ native textを利用可能
      │    └─ local extraction → Markdown
      └─ 読めないページがある
           └─ 対象ページだけrender → OCR → native textと統合
```

重要なのは、「このPDFはOCRするか」という文書単位の判定から、「どのページをどの根拠でOCRへ送るか」というroutingへ変わる点です。

たとえば100ページ中2ページだけがスキャン画像なら、98ページの文字層を維持したまま、2ページだけをOCRできます。逆に、壊れたfont encodingや文字の少ないページをnative extraction成功として扱うと、検索対象から内容が抜けます。分類結果だけでなく、`pages_needing_ocr`とページごとの理由を保存することが重要です。

## 3つのAPIを役割で分ける

### 1. `detect_pdf`は判定だけ行う

最初のgateでは、Markdownをまだ生成せず、分類とOCR候補ページを得ます。

```python
import pdf_inspector

result = pdf_inspector.detect_pdf("document.pdf")

print(result.pdf_type)
print(result.confidence)
print(result.page_count)
print(result.pages_needing_ocr)
print(result.ocr_reasons_by_page)
```

`pages_needing_ocr`は1始まりのページ番号です。後段のjob queueへ渡す場合は、0始まりのindexと混同しないようschemaに単位を明記します。

### 2. `process_pdf`はnative extractionとMarkdown化を行う

OCR不要のPDFは、次だけで分類、抽出、Markdown変換まで進められます。

```python
result = pdf_inspector.process_pdf("document.pdf")

if result.pdf_type == "text_based":
    markdown = result.markdown
else:
    print("OCR candidates:", result.pages_needing_ocr)
```

見出し、list、table、multi-columnのreading orderなどをPDF内の位置情報から組み立てます。ただし、Markdownが空でないことと、意味的に正しい順序であることは別です。後述するfixture testや自分の文書集合で確認します。

### 3. `process_pdf_with_ocr`は必要ページだけOCRする

v1.15.0のselective OCR entry pointです。

```python
result = pdf_inspector.process_pdf_with_ocr("document.pdf")

print("routed:", result.pages_routed_to_ocr)

for page in result.pages:
    print(
        page.page_number,
        page.provenance.source,
        page.provenance.ocr_confidence,
        page.provenance.warnings,
    )
```

各ページの`source`は`native`、`ocr`、`fused`のいずれかです。最終Markdownだけでなく、どの経路で作られたかというprovenanceを残せます。

`fused`は、利用可能なnative textを捨てずにOCR結果と統合したページです。検索結果の誤りを調べる際、「元PDFに文字がなかった」のか、「OCRが失敗した」のか、「統合処理で順序が変わった」のかを切り分けやすくなります。

## v1.15.0を固定して再現した結果

公式repositoryのv1.15.0をcloneし、tagのSHAがRelease記載のsource checkpointと一致することを確認しました。

```bash
git clone --depth 1 --branch v1.15.0 \
  https://github.com/firecrawl/pdf-inspector.git

git -C pdf-inspector rev-parse HEAD
```

出力は次でした。

```text
06a9bab6b3309309503f2db17851389cee094a62
```

次に、repositoryに含まれる2つのfixtureをPython package 1.15.0で処理しました。

```python
from pathlib import Path
import pdf_inspector

fixtures = Path("pdf-inspector/tests/fixtures")

for name in [
    "firecrawl_docs_tagged.pdf",
    "scan_with_native_header_text.pdf",
]:
    path = fixtures / name
    detected = pdf_inspector.detect_pdf(str(path))
    processed = pdf_inspector.process_pdf(str(path))

    print({
        "name": name,
        "type": detected.pdf_type,
        "confidence": round(detected.confidence, 3),
        "pages": detected.page_count,
        "pages_needing_ocr": detected.pages_needing_ocr,
        "markdown_chars": len(processed.markdown or ""),
    })
```

実行結果です。

```text
{
  'name': 'firecrawl_docs_tagged.pdf',
  'type': 'text_based',
  'confidence': 1.0,
  'pages': 7,
  'pages_needing_ocr': [],
  'markdown_chars': 11118
}
{
  'name': 'scan_with_native_header_text.pdf',
  'type': 'image_based',
  'confidence': 0.8,
  'pages': 1,
  'pages_needing_ocr': [1],
  'markdown_chars': 0
}
```

1つ目はnative extractionから11,118文字のMarkdownが生成され、冒頭は`# Firecrawl API Documentation`でした。2つ目はheaderにnative textがあるfixtureですが、本文を画像として判定し、1ページ目をOCR候補へ回しました。

この結果が示すのは、少量のnative textが存在するだけで文書全体を`text_based`にしないことです。一方、これは同梱fixtureでの動作確認であり、任意の帳票、縦書き日本語、手書き、低解像度scanに対する精度を証明するものではありません。

## Python wheelとRust CLIを混同しない

READMEにはPython APIとCLIの例がありますが、v1.15.0のPython wheelを隔離環境へ入れて確認したところ、`console_scripts` entry pointは空で、`pdf2md` executableは追加されませんでした。

```text
version 1.15.0
console_scripts []
```

`pdf2md`はRust側のbinary targetです。Python環境で使うなら、上記の`pdf_inspector` moduleをimportします。CLIを使う場合はRust crateを導入します。

```bash
cargo install pdf-inspector --version 1.15.0

pdf2md document.pdf --json
```

package managerが同じ名前でも、配布artifactに含まれるentry pointは異なります。production imageでは、実際に呼ぶbinaryまたはmoduleがinstall後に存在することをsmoke testへ入れます。

## OCR runtimeとmodelを明示的に管理する

selective OCRは「packageを入れれば、どの環境でも即座にOCRできる」という意味ではありません。公式Python documentでは、OCRされるページに次が必要だと説明されています。

- PDF pageを画像化するPDFium
- ONNX Runtimeのshared library
- PP-OCRv6 Smallの固定model set

Python wheel自体には、OCR model、PDFium、ONNX Runtimeは埋め込まれていません。cleanなPDFに対する`auto`処理は、それらをload・downloadしない設計です。

runtimeが標準のlibrary search pathにない場合は、公式手順に従って`PDFIUM_LIB_PATH`と`ORT_DYLIB_PATH`を設定します。初めてOCRへroutingされた際は、固定model setをdownloadし、checksumを検証します。

本番では、起動後に外部downloadを許すより、build時にartifactを取得・検証してwarm cacheまたは専用model directoryへ配置する方が予測可能です。

```python
result = pdf_inspector.process_pdf_with_ocr(
    "document.pdf",
    page_numbers=[1, 3],
    model_directory="/opt/models/pp-ocrv6-small",
    offline=True,
)
```

`offline=True`はnetwork accessを禁止します。必要なartifactが不足していれば、その場で黙って別経路へ進むのではなく、jobを失敗またはhosted fallback候補として扱います。

:::message alert
PDFは外部入力です。parserやOCR runtimeを、秘密情報や広いfilesystem権限を持たないcontainerで実行し、file size、page数、処理時間、memory、展開後画像sizeへ上限を設けてください。MIT licenseやTrending順位は、未信頼PDFの安全性を保証しません。
:::

## RAGへ入れる前にprovenanceを保存する

最終Markdownだけをvector storeへ入れると、後から抽出経路を追跡できません。最低限、documentとpageごとに次を保存します。

```python
record = {
    "document_sha256": source_sha256,
    "source_page": page.page_number,
    "extractor": "pdf-inspector",
    "extractor_version": "1.15.0",
    "source_commit": "06a9bab6b3309309503f2db17851389cee094a62",
    "route": page.provenance.source,
    "ocr_model": (
        page.provenance.ocr_model.name
        if page.provenance.ocr_model else None
    ),
    "warnings": page.provenance.warnings,
    "content": page.markdown,
}
```

`document_id`には、利用条件が許す範囲で元fileのhashを使えます。versionとrouteを持たせると、parser更新後にnative extractionだけ再実行する、OCR由来のpageだけ人手確認する、といった再処理ができます。

検索UIにも元pageへのlinkと`native` / `ocr`表示を出すと、利用者は回答の根拠を原文へ戻って確認できます。

## 作者公開benchmarkの読み方

公式READMEは、OpenDataLoaderの200 PDF corpusで、OCRを無効にしたlocal parser比較を公開しています。2026年7月31日にApple M4 Proで更新された表では、pdf-inspector 0.2.6について次の値が報告されています。

| 指標 | 作者公開値 |
| --- | ---: |
| Overall | 0.875 |
| Reading Order | 0.915 |
| Tables（TEDS） | 0.814 |
| Headings | 0.788 |
| 200 documentsの処理時間 | 0.470秒 |

速度はwarm-upを除外し、全corpusを逐次処理した5回の中央値です。予測、評価、chartは[再現用results branch](https://github.com/firecrawl/opendataloader-bench/tree/abi/pdf-parser-benchmark-results)で公開されています。

ただし、この記事では200 PDFのfull benchmarkを再実行していません。また、表の対象は0.2.6、今回の動作確認は1.15.0です。したがって「1.15.0でも同じscore・速度」とは断定できません。

採用評価では、公開値をそのまま自社SLAにせず、実際に扱う文書を層化します。

- native text / scan / mixed
- 日本語横書き / 縦書き / 英語
- 単一column / 複数column
- table / chart / 数式 / footnote
- 正常font / 壊れたencoding
- 暗号化 / 破損 / 巨大file

正解Markdownとpage routingの期待値を固定し、version更新ごとに回帰testします。

## 実運用のrouting例

API serverとOCR workerを分ける場合、次の流れにできます。

```text
upload
  ↓ size / page / MIME / timeout gate
quarantine storage
  ↓
detect_pdf
  ├─ text_based → process_pdf → quality checks
  └─ OCR候補あり → restricted OCR queue
                         ↓
                 process_pdf_with_ocr
                         ↓
                 provenance + warnings
  ↓
Markdown validation
  ├─ pass → chunking / indexing
  └─ fail → manual review / reject
```

quality checkでは、Markdownが空でないことだけでなく、次を確認します。

1. page数と抽出結果の対応が維持されているか
2. 文字数が急減したpageがないか
3. tableの列数やheaderが崩れていないか
4. OCR confidenceとwarningが閾値内か
5. source pageへ戻れる識別子があるか
6. password付き・破損PDFを正常扱いしていないか
7. parser timeoutを成功としてcacheしていないか

低confidenceをLLMで「補完」してはいけません。原文にない文字列をもっともらしく作ると、抽出失敗が検出しにくいデータ汚染へ変わります。

## 導入チェックリスト

- [ ] package versionとsource SHAを固定した
- [ ] Python APIとRust CLIの配布差を確認した
- [ ] native / OCR / fusedのprovenanceを保存する
- [ ] page番号の0始まり・1始まりをschemaで区別した
- [ ] OCR runtimeとmodel artifactをchecksum付きで管理する
- [ ] offline時の失敗・fallback方針を決めた
- [ ] 未信頼PDFを権限制限した環境で処理する
- [ ] file size、page数、時間、memoryへ上限を設けた
- [ ] 自社文書でreading order、table、縦書きを評価した
- [ ] version更新時に固定corpusで回帰testする
- [ ] Trendingや作者benchmarkだけで採用を決めない

## まとめ

pdf-inspector v1.15.0のselective OCRで重要なのは、OCR engineを追加したことだけではありません。PDF処理を、文書全体への一律変換から、ページごとの証拠を持つroutingへ変えられる点です。

固定版の実行では、text-basedな7ページのfixtureから11,118文字のMarkdownを生成し、native headerを持つ画像主体のfixtureは1ページ目をOCR候補として分離できました。

実運用では次の順に導入すると安全です。

1. `detect_pdf`で分類と候補pageを記録する
2. 読めるpageはnative extractionを優先する
3. 必要pageだけ制限されたOCR workerへ送る
4. `native` / `ocr` / `fused`と警告を保存する
5. 自社corpusで回帰testしてからindexへ入れる

AIへPDFを渡す前に、まず「どの文字が、どのpageから、どの処理で得られたか」を説明できるpipelineにしてください。

## 参考リンク

- [firecrawl/pdf-inspector](https://github.com/firecrawl/pdf-inspector)
- [v1.15.0 Release](https://github.com/firecrawl/pdf-inspector/releases/tag/v1.15.0)
- [Python API documentation](https://github.com/firecrawl/pdf-inspector/blob/v1.15.0/docs/python.md)
- [OCR runtime setup](https://github.com/firecrawl/pdf-inspector/blob/v1.15.0/docs/ocr-runtime.md)
- [Benchmarking guide](https://github.com/firecrawl/pdf-inspector/blob/v1.15.0/docs/benchmarking.md)
- [Reproducible benchmark results](https://github.com/firecrawl/opendataloader-bench/tree/abi/pdf-parser-benchmark-results)
- [GitHub Trending](https://github.com/trending?since=daily)
