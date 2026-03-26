---
title: "アーキテクチャパターン比較（VSA・CA）学習メモ"
type: "Tech"
description: "CAとVSAの思想的な違い・エンドポイントの配置論争・ETL移行との対比を整理した学習メモ。確立されていない議論については推論と事実を分けて記載。

・https://claude.ai/chat/4e2320b7-58fc-4658-9571-ad31331305bd"
tags: ["Architecture"]
date: "2026-03-26T00:00:00"
---

## 1. Vertical Slice Architecture（VSA）
### 提唱者と経緯
VSAはJimmy Bogardが2018年のブログ記事で提唱したアーキテクチャパターンです。BogardはAutoMapperやMediatRの作者でもあります。  
BogardはOnion Architectureを採用した長期プロジェクトで数ヶ月後に限界を感じ、CQRSとともにVertical Sliceへ移行したという経験を語っています。以降7〜8年間、あらゆる種類のシステムでVSAを採用し続けているとのことです。  
### 生まれた動機
> 「機能を追加するたびに多くの層を横断してファイルを追加しなければならない。UIを変更し、モデルにフィールドを追加し、バリデーションを修正する…これらは変更の軸に沿って結合されていない。」  
> — Jimmy Bogard  
動機は2つです。  
1. **開発が遅い**：1機能の変更が複数層に散らばる  
2. **層の維持がビジネス価値より優先されてしまう**：層の整合性を保つことが目的化する  
### CAとVSAの切り口の違い
```plain text
CA（技術的役割で水平に切る）:
Controllers / Services / Repositories / Domain

VSA（ユースケースで垂直に切る）:
Features/CreateTodo / Features/CompleteTodo / Features/DeleteTodo
```
Bogardの言葉：  
> 「スライス内の結合を最大化し、スライス間の結合を最小化せよ。」  
### VSAの理想構造
プレゼンテーション層とアプリケーション層がFeatureに統合され、**「1つの変更が1つのディレクトリに収まる」** 状態を目指します。  
```plain text
src/
  Features/
    Todos/
      Create/
        CreateTodo.cs     ← Endpoint + Command + Handler + Validator が同居
      Complete/
        CompleteTodo.cs
  Domain/                 ← 複数Featureをまたぐ共有エンティティとして残る
  Infrastructure/         ← 技術的関心事として残る
```
---
## 2. VSAのリスクと難しさ
### リスク①：境界線の引き方が難しい
VSAで最も判断が難しいのが「スライスの粒度をどこで区切るか」という問題です。  
```plain text
細かすぎる（スライスが爆発する）:
  Features/Todos/CreateTodo/
  Features/Todos/CreateTodoWithDueDate/     ← これは別スライスが必要か？

粗すぎる（結局CAと同じになる）:
  Features/Todos/                           ← Todosに全部詰め込む
    CreateTodo.cs / UpdateTodo.cs / DeleteTodo.cs
```
一般的なガイドラインとして、スライスは「ビジネス上の境界（境界付けられたコンテキスト）」に沿って切ることが推奨されています。正解がないため、チームの判断力と経験に依存しやすい点がリスクです。  
### リスク②：スライス間のコード重複
各スライスが独立しているため、似たような処理を複数のスライスで書き直す状況が発生しやすくなります。Bogardは「最小化せよ（minimize）であってゼロにせよとは言っていない」とし、重複が証明されてからリファクタリングする姿勢を推奨しています。  
### リスク③：チームの規律への依存
VSAはCAほど「型」がありません。「スライス間の結合を最小化する」という原則の解釈がチームによってばらつき、規律なく実装すると最終的に「大きな泥の塊（Big Ball of Mud）」になるリスクがあります。  
---
## 3. エンドポイントはプレゼン層か、Feature層か
これはVSAとCAの間で最も論争になりやすいポイントです。確立したベストプラクティスはありませんが、議論の傾向は整理できます。  
### VSA陣営：エンドポイントはFeatureに属する
Bogard本人がこう定義しています：  
> 「この手法では、フロントエンドからバックエンドまで、すべての関心事をカプセル化してグループ化する。n層アーキテクチャから層をまたぐゲートやバリアを取り除き、変更の軸に沿って結合する。」  
この定義に従えば、エンドポイント（HTTPの窓口）もFeatureの一部であり、Featureフォルダに同居するのが自然です。実際、以下の実装例はエンドポイントをFeatureに同居させることを前提として示しています：  
- nadirbad / VerticalSliceArchitecture テンプレート（GitHub）  
- DEV Community（Cristian Sifuentes）  
- antondevtips.com（VSA構造の4パターン）  
- adrianbailador.github.io  
- Medium（Mian Muhammad Mudarrib）  
また **Milan自身がVSAを解説した別のブログ記事**（Pragmatic Clean Architectureのサンプルとは別）では、エンドポイントをFeature内の静的クラスに同居させる構造を示しています。  
```c#
// Milan Jovanovic – VSA解説記事より
public static class CreateProduct
{
    public record Request(string Name, decimal Price);
    public record Response(int Id, string Name, decimal Price);

    public class Endpoint : IEndpoint
    {
        public void MapEndpoint(IEndpointRouteBuilder app)
            => app.MapPost("products", Handler).WithTags("Products");

        public static IResult Handler(Request request, AppDbContext context)
        {
            var product = new Product { Name = request.Name, Price = request.Price };
            context.Products.Add(product);
            context.SaveChanges();
            return Results.Ok(new Response(product.Id, product.Name, product.Price));
        }
    }
}
```
### CA陣営：プレゼン層はHTTPの知識を閉じ込める場所として分離すべき
Robert C. Martinが提唱したClean Architectureの原則の1つに**「フレームワーク独立性」**があります。Application層はフレームワークに依存すべきではないという考え方です。  
エンドポイントをFeatureに統合すると、同じFeatureという単位の中に`IResult`・`Results.Ok`・`MapPost`といったASP.NET Core固有の型が混在します。ハンドラー自体がASP.NET Coreに依存するわけではありませんが、Featureとして一体化して管理されることで「フレームワーク非依存」という原則との方向性が逆行します。  
> ⚠️ **「だからMilanがプレゼン層を分けた」という解釈はCA原則からの推論であり、Milanが明言しているわけではありません。要確認。**  
### Milanが2つの異なる構造を示している事実
注目すべき点として、Milanは同じVSAというテーマに対して2つの異なる構造を示しています。  
| 記事・作品                             | エンドポイントの配置                        |
| --------------------------------- | --------------------------------- |
| VSA解説ブログ記事                        | **Featureに同居**（静的クラス内にEndpoint）   |
| Pragmatic Clean Architecture サンプル   | **プレゼン層として分離**（Web.Api/Endpoints）   |
これ自体が「どちらが正解かは文脈による」ということを示しています。  
### NDepend（Patrick Smacchia）の整理
> 「一つのサイズですべてに合う解決策はない。レイヤーはより多くの結合と硬直性をもたらし、スライスはより多くの重複と一貫性の欠如をもたらす。水平方向と垂直方向のどちらを優先するかを決断し、それに伴う特定の問題に対処することが必要だ。」  
### 現時点での傾向まとめ
確立したベストプラクティスはなく、議論は継続中です。傾向として：  
- **純粋なVSAを採用する場合**：エンドポイントもFeatureに含める  
- **CAの原則を維持しつつVSAを取り入れる場合**：プレゼン層を残してエンドポイントを分離する  
- **判断基準**：「このロジックを別の接点（CLI、gRPC）から呼ぶ可能性があるか」。ある場合はフレームワーク依存を分離しておく価値がある  
---
## 4. CAとVSAの比較
### 向いているケース
|             | Clean Architecture | Vertical Slice Architecture |
| ----------- | ------------------ | --------------------------- |
| **向いている場面**   | 複雑なビジネスルールを持つシステム   | ユースケースが多く独立して変化するシステム       |
| **変更の単位**   | 層（横断的な変更）          | Feature（局所的な変更）             |
| **テスト戦略**   | ユースケース・ドメイン単位      | スライス単位の統合テスト                |
| **学習コスト**   | 依存関係ルールの理解が必要      | 層の概念が不要で直感的                 |
| **リスク**     | 層の維持が目的化しやすい       | 境界の判断・コード重複・チームの規律          |
### 同じ問題を解いている
CodeOpinionのDerek Comarは「VSAはコヒージョン（凝集度）についてのパターンで、CAは依存関係の方向についてのパターン。両者は直交する関心事であって相互排他ではない」と整理しています。  
### よくある誤解
| 誤解                             | 実際                               |
| ------------------------------ | -------------------------------- |
| スライス間でコードを共有してはいけない            | 「最小化せよ」であって「ゼロにせよ」ではない           |
| MediatRが必須                     | VSAはアーキテクチャパターンであってライブラリの選択ではない   |
| CAのフォルダをFeatureに並び替えるだけでVSAになる   | Bogardは「過剰設計のゴミを並べ替えただけ」と批判している   |
| VSAはCAより簡単                     | 境界の判断・凝集度・結合度の理解が必要で、むしろ難しい側面もある   |
---
## 5. Milan JovanovicのCA+VSAハイブリッド
### 構造の特徴
Pragmatic Clean Architectureサンプルは、CAの層構造を維持しながらもApplicationフォルダ内をFeatureで切るハイブリッド構成です。  
```plain text
src/
  Web.Api/
    Endpoints/
      Todos/        ← プレゼン層はCAの構造のまま（意図的に残す）
        Create.cs
  Application/
    Todos/
      Create/       ← アプリ層をFeatureで切る（VSA的）
        CreateTodoCommand.cs
        CreateTodoCommandHandler.cs
  Domain/
  Infrastructure/
```
Milanの立場：  
> 「複雑なビジネスルールを持つスライスにはCAとリッチドメインモデルを使う。シンプルなスライスにはトランザクションスクリプトで十分だ。」  
---
## 6. 層の数は手段であって目的ではない
> 「重要なのは関連するコードを一緒に置くことだ。各スライスが独立して進化できることが重要。」  
> — Jimmy Bogard  
VSAはCAへのアンチテーゼとして生まれましたが、以下は否定していません。  
- ドメインモデルを持つこと  
- インフラを分離すること  
- DRY原則（ただし、スライス間の共有は最小化する）  
---
## 7. ETL移行との対比
|        | NotionConverter ETL移行          | WebAPI CA→VSA                  |
| ------ | ------------------------------ | ------------------------------ |
| 崩した層   | Application層、Domain層           | Application層、Presentation層     |
| 残した層   | Infrastructure                 | Infrastructure、Domain（共有概念として）   |
| まとめた単位   | 処理ステージ（Extract/Transform/Load）   | ユースケース（Feature）                |
| 共通する本質   | 技術的な層で切るのをやめて関心事で切る            | 同左                             |
---
## 参考ソース
### VSA全般
| 内容                                  | URL                                                                                                             |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| VSA原典（Jimmy Bogard・2018年）           | https://www.jimmybogard.com/vertical-slice-architecture/                                                        |
| VSA公式サイト                            | https://verticalslicearchitecture.com/                                                                          |
| VSAの誤解を解説（CodeOpinion）              | https://codeopinion.com/vertical-slice-architecture-myths-you-need-to-know/                                     |
| VSA vs CA（NDepend・Patrick Smacchia）   | https://blog.ndepend.com/vertical-slice-architecture-in-asp-net-core/                                           |
| VSA vs CA（Mehmet Ozkaya）            | https://mehmetozkaya.medium.com/vertical-slice-architecture-and-comparison-with-clean-architecture-76f813e3dab6   |
### エンドポイントをFeatureに含める実装例群
| 内容                              | URL                                                                                                                                          |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| VSA実装テンプレート（nadirbad）           | https://github.com/nadirbad/VerticalSliceArchitecture                                                                                        |
| VSAの4パターン（antondevtips）         | https://antondevtips.com/blog/vertical-slice-architecture-the-best-ways-to-structure-your-project                                            |
| VSA解説・エンドポイント同居（DEV Community）   | https://dev.to/cristiansifuentes/vertical-slice-architecture-in-net-from-n-tier-layers-to-feature-slices-4iha                                |
| VSA解説・エンドポイント同居（adrianbailador）   | https://adrianbailador.github.io/blog/47-vertical-slice-architecture/                                                                        |
| VSA解説・エンドポイント同居（Medium）         | https://medium.com/@mmudarrib2807/mastering-vertical-slice-architecture-in-net-core-a-clean-scalable-approach-to-web-api-design-6f6813488d17   |
| Milan自身のVSA解説（エンドポイント同居版）       | https://www.milanjovanovic.tech/blog/vertical-slice-architecture-structuring-vertical-slices                                                 |
### エンドポイントをプレゼン層として分離する実装例
| 内容                                                  | URL                                                              |
| --------------------------------------------------- | ---------------------------------------------------------------- |
| Milan Jovanovic – Pragmatic Clean Architecture（分離版）   | https://www.milanjovanovic.tech/blog/vertical-slice-architecture   |
### CA原則（フレームワーク独立性）
| 内容                                   | URL                                                                          |
| ------------------------------------ | ---------------------------------------------------------------------------- |
| Clean Architecture（Robert C. Martin）   | https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html   |

