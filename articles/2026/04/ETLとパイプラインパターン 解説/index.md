---
title: "ETLとパイプラインパターン 解説"
type: "Tech"
description: "・https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd"
date: "2026-04-02T00:00:00"
---

## 目次
1. [ETLとは何か](https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd#1-etl%E3%81%A8%E3%81%AF%E4%BD%95%E3%81%8B)  
2. [ETLの各フェーズ詳解](https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd#2-etl%E3%81%AE%E5%90%84%E3%83%95%E3%82%A7%E3%83%BC%E3%82%BA%E8%A9%B3%E8%A7%A3)  
3. [ETLの歴史と進化](https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd#3-etl%E3%81%AE%E6%AD%B4%E5%8F%B2%E3%81%A8%E9%80%B2%E5%8C%96)  
4. [ELT：ETLの派生と現代的な変化](https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd#4-elt%E3%81%88tl%E3%81%AE%E6%B4%BE%E7%94%9F%E3%81%A8%E7%8F%BE%E4%BB%A3%E7%9A%84%E3%81%AA%E5%A4%89%E5%8C%96)  
5. [パイプラインパターン（Pipes and Filters）](https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd#5-%E3%83%91%E3%82%A4%E3%83%97%E3%83%A9%E3%82%A4%E3%83%B3%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3pipes-and-filters)  
6. [ETLとパイプラインパターンの関係](https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd#6-etl%E3%81%A8%E3%83%91%E3%82%A4%E3%83%97%E3%83%A9%E3%82%A4%E3%83%B3%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3%E3%81%AE%E9%96%A2%E4%BF%82)  
7. [C#での実装例](https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd#7-c%E3%81%A7%E3%81%AE%E5%AE%9F%E8%A3%85%E4%BE%8B)  
8. [メリットとデメリット](https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd#8-%E3%83%A1%E3%83%AA%E3%83%83%E3%83%88%E3%81%A8%E3%83%87%E3%83%A1%E3%83%AA%E3%83%83%E3%83%88)  
9. [Clean Architectureとの比較](https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd#9-clean-architecture%E3%81%A8%E3%81%AE%E6%AF%94%E8%BC%83)  
10. [参考ソース](https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd#10-%E5%8F%82%E8%80%83%E3%82%BD%E3%83%BC%E3%82%B9)  
---
## 1. ETLとは何か
**ETL（Extract, Transform, Load）** とは、データをソースシステムから抽出し、変換し、出力先に格納するという3フェーズのデータ処理プロセスです。  
```plain text
[Source] ──Extract──▶ [Staging] ──Transform──▶ [Staging] ──Load──▶ [Destination]
```
ETLはもともとデータウェアハウスへのデータ統合を目的に発展しましたが、現在ではファイル変換ツール、API連携、バッチ処理など広範なシステムに応用されている概念です。  
---
## 2. ETLの各フェーズ詳解
### Extract（抽出）
ソースシステムからデータを取り出すフェーズです。  
**重要原則：ソースデータには手を加えない。生データのまま次のフェーズに渡す。**  
ETLプロセスの最初のステップでは、データがソースからステージングエリアに抽出されます。抽出されたデータはバリデーションルールに対してチェックされ、早期にエラーを検出することで後続フェーズのコストを下げます。  
Extractに共通するアプローチとして、フルロード（全データを毎回取得）とインクリメンタルロード（差分のみ取得）があります。大規模システムでは、最終取得日時を記録して差分だけを処理するインクリメンタルロードが一般的です。  
### Transform（変換）
抽出したデータを出力先の形式・ルールに合わせて加工するフェーズです。  
データ変換ステージでは、一連のルールまたは関数が抽出されたデータに適用され、最終ターゲットへのロードに向けた準備が行われます。変換の重要な機能はデータクレンジングであり、「適切な」データのみをターゲットに渡すことを目的とします。  
主な変換処理の例：  
- データ型の変換・正規化  
- フォーマット変換（例：NotionブロックデータをMarkdownへ）  
- フィルタリング・条件判定  
- エンリッチメント（外部データの付加）  
- 集約・結合・重複排除  
### Load（格納）
変換済みデータを出力先に書き込むフェーズです。  
**ベストプラクティス：データは最終的な状態に完全に準備してからロードする。ロード中に変換やクレンジングを行うべきではありません。**  
ロードには全置換（洗い替え）と差分追記（アペンド）があります。べき等性（同じ処理を何度実行しても結果が変わらない性質）を確保しておくと、障害時の再実行が安全になります。  
---
## 3. ETLの歴史と進化
### 1970年代：起源
ETLの概念は1970年代、大規模な集中型データベースへの移行が進んだ際に生まれました。当時はデータを統合・ロードするためのプロセスとして自然発生的に形成されたもので、GoFのデザインパターン（1994年）のように「誰かが提唱した」ものではなく、業界の実践の中で形成された概念です。  
この時代は構造化プログラミングの全盛期で、「順番に処理する」という手続き的な流れがそのままExtract→Transform→Loadに対応していました。  
### 1980〜90年代：データウェアハウスの台頭
1980年代後半から1990年代にかけて、リレーショナルデータベースの普及とビジネスインテリジェンス（BI）の需要拡大により、ETLは本格的に使われ始めました。この時期、ストレージは高価でコンピューティングリソースも限られていたため、「ロード前にできる限り変換を済ませる」というETLのアプローチが合理的な選択でした。  
### 2000年代：ETLツールの成熟
Informatica、IBM DataStage、Microsoft SSISなど、専用のETLツールが製品として成熟しました。大規模なバッチ処理が前提で、夜間バッチでデータウェアハウスを更新するパターンが一般的でした。  
### 2010年代〜：クラウドとELTへの転換
2017年前後、Snowflake・Amazon Redshift・Google BigQueryなどのクラウドネイティブなデータウェアハウスの登場により、状況が一変します。ストレージが安価になり、クラウド側の処理能力が向上したことで、「ロードしてからクラウド上で変換する」ELTが主流になっていきました。  
---
## 4. ELT：ETLの派生と現代的な変化
### ELTとは
ELT（Extract, Load, Transform）はETLの変形で、変換のタイミングを逆転させます。  
```plain text
ETL:  Extract → Transform → Load   （ロード前に変換）
ELT:  Extract → Load → Transform   （ロード後に変換）
```
ELTでは抽出したデータをそのまま出力先にロードし、クラウドデータウェアハウスの処理能力を使って変換します。これによりステージング環境の必要性が排除され、生データが保持されるため後から異なる分析に使いまわすことができます。  
### ETLとELTの比較
| 比較軸      | ETL           | ELT            |
| -------- | ------------- | -------------- |
| 変換タイミング   | ロード前          | ロード後           |
| 変換場所     | 専用の変換エンジン     | 出力先のデータウェアハウス   |
| 向いているデータ   | 構造化データ        | 非構造化・半構造化データも可   |
| ストレージ要件   | ステージングエリアが必要   | 生データをそのまま保存    |
| 主な用途     | オンプレミス・レガシー環境   | クラウドネイティブ環境    |
### Reverse ETL
近年ではReverse ETLという概念も登場しています。通常のETLが「運用システム→分析システム」へデータを流すのに対し、Reverse ETLは「分析システム→運用システム」へ加工済みデータを戻します。データウェアハウスで得たインサイトをCRMや広告ツールなどのビジネスツールに反映させるために使われます。  
---
## 5. パイプラインパターン（Pipes and Filters）
### 概要
**Pipes and Filters** は、複雑なデータ処理タスクを小さな独立したコンポーネント（Filter）の連鎖に分解するアーキテクチャパターンです。POSA（Pattern-Oriented Software Architecture, 1996年）で体系化されました。  
> 「複雑な処理を実行するタスクを、個別に再利用可能な一連の要素に分解する。これにより処理を実行するタスク要素を独立してデプロイ・スケールさせることができ、パフォーマンス、スケーラビリティ、再利用性を向上できる。」  
> — Microsoft Azure Architecture Center  
### 構成要素
| 要素               | 役割                             |
| ---------------- | ------------------------------ |
| **Pump（Source）**   | データの入力元。パイプラインへデータを供給する        |
| **Filter**       | データを受け取り、変換・加工して次のフィルターに渡す処理単位   |
| **Pipe**         | フィルター間のデータの通路                  |
| **Sink**         | パイプラインの最終出力先                   |
```plain text
[Pump] ──pipe──▶ [Filter1] ──pipe──▶ [Filter2] ──pipe──▶ [Filter3] ──pipe──▶ [Sink]
```
### フィルターの特性
各フィルターは特定のデータ操作を実行する責務を持ちます。重要な設計原則は、**フィルターがお互いの存在を知らないこと**です。フィルターはパイプライン内の自分の位置を直接知らない状態で設計されます。  
フィルターがデータに対して行う操作は大きく3種類です：  
- **Enrich（充実化）**：外部データを付加する  
- **Refine（精製）**：不要なデータを除去・絞り込む  
- **Transform（変換）**：形式や構造を変える  
### Unixパイプとの類似
このパターンはUnixシェルのパイプ構文から着想を得ています。  
```bash
cat file.txt | grep "pattern" | sort | uniq
```
各コマンドが独立しており、入力を受け取って出力を次に渡すという構造がPipes and Filtersそのものです。  
---
## 6. ETLとパイプラインパターンの関係
ETLは**「何をするか」を表すデータ処理の概念**であり、Pipes and Filtersは**「どう構造化するか」を表すアーキテクチャパターン**です。両者は直交する概念ですが、相性が極めて良いです。  
```plain text
ETL概念:         Extract          Transform          Load
                    ↕                 ↕                ↕
P&Fパターン:   [ExtractFilter]──▶[TransformFilter]──▶[LoadFilter]
```
相性が良い理由は、ETLの3フェーズがそれぞれ「単一責任」「疎結合」「独立したテスト可能性」を求めており、これがPipes and Filtersの設計思想と完全に一致するからです。  
### 各パラダイムとの相性
| 時代          | パラダイム        | ETL/パイプラインとの相性           |
| ----------- | ------------ | ------------------------ |
| 1970〜80年代   | 構造化プログラミング   | ◎ 手続き的な流れがそのまま対応         |
| 1990〜2000年代   | オブジェクト指向     | △ 「データと振る舞いの一体化」とやや相性が悪い   |
| 2010年代〜     | 関数型・マルチパラダイム   | ◎ 純粋関数の合成がパイプラインと同構造     |
C#のLINQや`async/await`は関数型的な要素であり、パイプライン処理との相性が良いです。  
---
## 7. C#での実装例
### インターフェース定義
```c#
// パイプライン上の1つの処理段階を表すインターフェース
public interface IPipe<TIn, TOut>
{
    Task<TOut> ProcessAsync(TIn input);
}
```
### フィルターの実装例（Notion→Markdown変換を想定）
```c#
// Extract: Notionからページデータを取得する
public class ExtractFilter(INotionPageReader reader) : IPipe<string, List<ExtractedPage>>
{
    public async Task<List<ExtractedPage>> ProcessAsync(string databaseId)
    {
        // ソースからの抽出のみを担当。変換しない。
        var pages = await reader.GetPagesForPublishingAsync(databaseId);
        return pages.Select(p => new ExtractedPage(p)).ToList();
    }
}

// Transform: ページデータをMarkdownに変換する
public class TransformFilter(IMarkdownConverter converter) : IPipe<List<ExtractedPage>, List<TransformedPage>>
{
    public async Task<List<TransformedPage>> ProcessAsync(List<ExtractedPage> pages)
    {
        // 変換のみを担当。I/Oしない。
        var results = new List<TransformedPage>();
        foreach (var page in pages)
            results.Add(await converter.ConvertAsync(page));
        return results;
    }
}

// Load: Markdownをファイルシステムに書き出す
public class LoadFilter(IFileSystem fileSystem) : IPipe<List<TransformedPage>, int>
{
    public async Task<int> ProcessAsync(List<TransformedPage> pages)
    {
        // 格納のみを担当。変換しない。
        foreach (var page in pages)
            await fileSystem.WriteAllTextAsync(
                Path.Combine(page.OutputDirectory, "index.md"),
                page.Markdown,
                new UTF8Encoding(false));
        return pages.Count;
    }
}
```
### パイプラインの組み立て
```c#
public class Pipeline<TIn, TOut>
{
    private readonly Func<TIn, Task<TOut>> _execute;

    private Pipeline(Func<TIn, Task<TOut>> execute) => _execute = execute;

    public static Pipeline<TIn, TOut> Of(IPipe<TIn, TOut> pipe)
        => new(input => pipe.ProcessAsync(input));

    public Pipeline<TIn, TNext> Then<TNext>(IPipe<TOut, TNext> next)
        => new(async input => await next.ProcessAsync(await _execute(input)));

    public Task<TOut> RunAsync(TIn input) => _execute(input);
}

// 使用例
var exportedCount = await Pipeline
    .Of(new ExtractFilter(notionReader))
    .Then(new TransformFilter(markdownConverter))
    .Then(new LoadFilter(fileSystem))
    .RunAsync(databaseId);
```
各フィルターが「受け取って、処理して、渡す」という単一責任に集中しており、依存関係が一方向に流れています。  
---
## 8. メリットとデメリット
### メリット
- **テスト容易性**：各フィルターが純粋な入出力変換として設計されれば、依存なしで単体テスト可能  
- **再利用性**：フィルターを別のパイプラインに組み込める  
- **拡張性**：新しいフィルターをパイプラインに挿入・置換できる  
- **並列化**：独立したフィルターは並列実行できる  
ETLアプリケーションには3種類の並列処理があります：  
- **データ並列**：単一ファイルを分割して並列アクセス  
- **パイプライン並列**：同じデータストリームで複数コンポーネントを同時実行  
- **コンポーネント並列**：異なるデータストリームに対して複数プロセスを同時実行  
### デメリット
- フィルター間の状態共有が困難（設計上の意図でもある）  
- エラーハンドリングの設計が煩雑になりやすい  
- フィルター数が増えると全体の流れが追いにくくなる  
- データが多数のフィルターを通過する場合、追加の処理がレイテンシーを増加させる  
---
## 9. Clean Architectureとの比較
ETL的な変換ツールにClean ArchitectureとETL/Pipelineパターンのどちらが適しているかを整理します。  
| 観点              | Clean Architecture  | Pipes and Filters (ETL) |
| --------------- | ------------------- | ----------------------- |
| **向いているユースケース**   | ビジネスルールが複雑なアプリケーション   | データ変換・バッチ処理             |
| **コードの流れ**      | 層をまたぐ依存関係の制御が主眼     | データが一方向に流れることが前提        |
| **テスト戦略**       | ユースケース単位のテスト        | フィルター単位の単体テスト           |
| **拡張ポイント**      | インターフェースによる差し替え     | フィルターの追加・置換             |
| **オーバーヘッド**     | 小規模ツールには重い          | 処理の性質に直接対応              |
**判断基準：**  
```plain text
処理の本質が「データの流れ」    → Pipes and Filters
処理の本質が「ビジネスルール」  → Clean Architecture
```
**NotionMarkdownConverterの場合**  
「NotionデータをMarkdownに変換して出力する」というETLそのものの処理であるため、パイプラインパターンがアーキテクチャの主軸として機能します。ただし、ブロックタイプごとの変換処理（戦略パターン）のような詳細設計レベルではSOLID原則が依然として有効です。  
実際の移行後の構造：  
```plain text
src/
  Extract/               ← Extractステージ
    NotionPageExtractor.cs
    PageExportEligibilityChecker.cs
  Transform/             ← Transformステージ
    NotionPageTransformer.cs
    Converters/
    Strategies/
  Load/                  ← Loadステージ
    NotionPageLoader.cs
  Pipeline/              ← オーケストレーター
    NotionExportPipeline.cs
  Infrastructure/        ← 外部依存（層として残す）
  Shared/                ← 共有モデル・ユーティリティ
```
---
## 10. 参考ソース
| 内容                                                         | URL                                                                                                    |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Extract, transform, load（Wikipedia）                        | https://en.wikipedia.org/wiki/Extract,_transform,_load                                                 |
| ETL解説（AWS）                                                 | https://aws.amazon.com/what-is/etl/                                                                    |
| ETL解説（Microsoft Azure）                                     | https://learn.microsoft.com/en-us/azure/architecture/data-guide/relational-data/etl                    |
| Pipes and Filtersパターン（Microsoft Azure Architecture Center）   | https://learn.microsoft.com/en-us/azure/architecture/patterns/pipes-and-filters                        |
| ETLからELTへの進化（Medium）                                       | https://medium.com/@andymadson/the-evolution-of-data-pipelines-from-etl-to-elt-and-beyond-2e04deac7062   |
| ELT詳解（Databricks）                                          | https://www.databricks.com/blog/what-is-elt                                                            |
| ELT詳解（dbt Labs）                                            | https://www.getdbt.com/blog/extract-load-transform                                                     |
| ETLとELTの比較（dbt Labs）                                       | https://www.getdbt.com/blog/etl-tools-data-pipeline-architecture                                       |
| ELT詳解（Snowflake）                                           | https://www.snowflake.com/en/fundamentals/understanding-extract-load-transform-elt/                    |
| POSA原典（Pipes and Filtersパターンの原書PDF）                        | https://john.cs.olemiss.edu/~hcc/docs/Patterns/Pipes/Pipes.pdf                                         |

