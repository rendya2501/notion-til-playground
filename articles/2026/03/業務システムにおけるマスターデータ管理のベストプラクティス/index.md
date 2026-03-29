---
title: "業務システムにおけるマスターデータ管理のベストプラクティス"
type: "Tech"
description: "・https://claude.ai/chat/0eba869c-5ee9-4c85-9076-a83209b1eca0
・https://gemini.google.com/app/35f79f53897dfafc"
tags: ["Architecture"]
date: "2026-03-28T00:00:00"
---

## はじめに
業務アプリケーションで大分類・中分類・小分類のような階層構造を持つマスターデータを扱う場合、「削除」と「名称変更」の設計は見落とされがちながら、後から大きな問題を引き起こす領域です。  
本記事では以下の3点について、エンタープライズ観点からのベストプラクティスを整理します。  
1. **親データの削除をどう扱うか**  
2. **名称変更時に過去データとの整合性をどう守るか**  
3. **過去データの遡及修正はどう扱うべきか**  
## 1. 階層データの削除設計
### 代表的な4パターン
### ① 論理削除（推奨）
実データは残し、「削除済み」状態を管理する方法です。業務アプリでは最も一般的かつ推奨される手法です[^1]。  
論理削除の実装には「削除フラグ」と「ステータス管理」の2種類があります。詳細は後述します。  
- **メリット：** 誤操作からの復旧が容易。過去の統計データとの整合性が保ちやすいです  
- **デメリット：** 全クエリに削除済みを除外する条件が必要になります  
EF Core では `HasQueryFilter` でグローバルフィルターを設定することで、書き忘れのリスクを緩和できます。  
```c#
// Microsoft Learn 公式のソフトデリート実装例[^1]
public class Blog
{
    public int Id { get; set; }
    public bool IsDeleted { get; set; }
    public string Name { get; set; }
}

modelBuilder.Entity<Blog>().HasQueryFilter(b => !b.IsDeleted);
```
### ② アーカイブテーブル方式
削除対象のレコードを専用の `*_archive` テーブルへ移動する方法です。  
- **メリット：** アクティブなテーブルが常に軽量。WHERE句が不要になります。**ユニーク制約の問題も自然に解消されます**（後述）  
- **デメリット：** 復元処理やスキーマ管理が複雑になります  
### ③ 削除禁止（Validation Guard）
子データが存在する場合は親を削除できないよう、アプリケーション側でブロックします。  
- **メリット：** データ整合性が最も堅牢です  
- **デメリット：** 深い階層の場合、末端から手動で消す作業がユーザーのストレスになります  
### ④ カスケード削除（ON DELETE CASCADE）
DB制約で親削除時に子・孫を物理削除します。  
**業務システムでの採用は基本的に避けるべきです。** トランザクションデータから参照されている場合は外部キー制約エラーで削除が失敗し、誤操作時の影響も壊滅的です。  
### 論理削除の実装：削除フラグ vs ステータス管理
`is_deleted` フラグと `Status` enum はどちらも有効な実装です。**選択の基準は「ドメインに複数の意味ある状態が存在するか」です。**  
| 観点      | 削除フラグ（`is_deleted`） | ステータス管理（`Status` enum） |
| ------- | ------------------- | ---------------------- |
| 実装コスト   | 低い                  | やや高い                   |
| 表現力     | 「削除された/されていない」の二値   | ビジネスの意図をコードで表現できる      |
| 拡張性     | 状態が増えた場合にカラム追加が必要   | enum 値の追加で対応できる        |
| 適している場面   | 「有効/無効」の二値で十分な場合    | 複数の意味ある状態がビジネス上存在する場合   |
「廃止」「一時停止」「審査中」など複数の意味ある状態がビジネス上存在するなら Status enum が自然です。単純に「使う/使わない」の二値で十分なら `is_deleted` フラグで問題ありません。  
DDD の Eric Evans が説く「ドメインの概念を明示的にせよ」という原則は[^2]、フラグを禁止しているのではなく「ビジネスに存在する状態を正直にモデリングせよ」という意味です。`is_deleted` を `Status` enum に変えれば設計が改善されるわけではなく、**ドメインに複数の状態が実在するなら enum で表現する方が自然**ということです。  
いずれの場合も `deleted_at`（削除日時）と `deleted_by`（削除者）の audit 列を合わせて持たせることで、「いつ・誰が」の情報を補完できます。  
### 削除フラグを使う場合の実装例
```c#
public class Category
{
    public bool IsDeleted { get; private set; }
    public DateTime? DeletedAt { get; private set; }
    public string? DeletedBy { get; private set; }

    public void Delete(string deletedBy)
    {
        IsDeleted = true;
        DeletedAt = DateTime.UtcNow;
        DeletedBy = deletedBy;
    }
}
```
### ステータス管理を使う場合の実装例
```c#
public enum CategoryStatus
{
    Draft,     // 下書き（未公開）
    Active,    // 有効
    Obsolete   // 廃止
}
```
### ユニーク制約の問題と解決策
論理削除（フラグ・ステータスいずれも）が抱える問題として、**ユニーク制約が機能しなくなる**点があります。  
たとえばメールアドレスにユニーク制約を設けている場合、`is_deleted = true` のレコードが残っているだけで、同じメールアドレスでの再登録ができなくなります。ただし、この問題が発生するシナリオはケースによって性質が異なり、対応策も変わります。  
| ケース              | 内容                              | 推奨対応                |
| ---------------- | ------------------------------- | ------------------- |
| 他人との衝突           | 退会済みユーザーのメールアドレスに別ユーザーが登録しようとする   | 部分インデックス            |
| 同一エンティティの再登録     | 廃止した商品を同じコード・名称で復活させたい          | Reactivate（論理削除を解除）   |
| ビジネス的に「別物」として再作成   | 過去の商品とは切り離して新しく作りたい             | 新しいIDで新規作成          |
### ① 部分インデックス（Partial Unique Index）— 他人との衝突に有効
DBレベルでアクティブなレコードのみを対象にしたユニーク制約を設定します。  
```sql
-- アクティブなレコードにのみユニーク制約を設ける（PostgreSQL / SQL Server）
CREATE UNIQUE INDEX ux_users_email_active
    ON users (email)
    WHERE is_deleted = false;

-- ステータス管理の場合
CREATE UNIQUE INDEX ux_users_email_active
    ON users (email)
    WHERE status = 'Active';
```
### ② Reactivate（再有効化）— 同一エンティティの復活に有効
廃止した商品を「同じコードで再登録したい」というケースは、新規作成ではなく**既存の論理削除レコードを復活（Reactivate）させる**のが設計として自然です。「新しく作りたい」という要件があるとしても、それがビジネス的に「同じ商品の再開」であれば Reactivate が適切です。  
```c#
public class Product
{
    public CategoryStatus Status { get; private set; }

    public void Reactivate()
    {
        if (Status != CategoryStatus.Obsolete)
            throw new InvalidOperationException("廃止済み以外は再有効化できません");

        Status = CategoryStatus.Active;
    }
}
```
### ③ アーカイブテーブル方式
inactive なレコードを別テーブルに移すため、アクティブテーブルのユニーク制約が自然に機能するようになります。  
### ④ アプリケーション側チェック
既存の Active レコードを事前確認してから INSERT する方法です。同時アクセス時のレースコンディションリスクがあるため、データ量や要件を考慮して採用を判断してください。  
### 台帳（マスター）画面の正しい設計
**「台帳画面に削除ボタンを置かず、ステータス変更にする」** のが広く普及しているエンタープライズの設計慣行です。廃止や無効化を「削除」ではなくライフサイクルの状態遷移として扱う考え方は、大規模 ERP システム（SAP、Oracle EBS 等）でも一般的に採用されています。  
| 画面        | 設計                                     |
| --------- | -------------------------------------- |
| 一覧画面      | デフォルトは有効なデータのみ表示。「廃止済みを含める」チェックボックスを用意   |
| 編集画面      | 「この項目を無効にする」ボタンを設置                     |
| 物理削除の許可条件   | 参照ゼロ かつ Draft 状態の場合のみ許可                |
### 削除前の2段階チェック
```plain text
ユーザーが「削除」を要求
        ↓
トランザクションデータからの参照チェック
        ↓
  参照あり ──→ 物理削除をブロック → 廃止（論理削除）へ
        ↓
  参照なし かつ 作成直後 ──→ 物理削除を許可
```
### 「作成直後」の具体的な条件
| 条件        | 内容                                |
| --------- | --------------------------------- |
| 参照ゼロ（最重要）   | トランザクション系テーブルからの外部キー参照が1件もないこと    |
| ステータス     | 一度も Active になっていないこと（Draft 状態のまま）   |
| 時間窓（任意）   | 作成から24時間以内、または同一営業日内など            |
実務では「参照ゼロ」の条件単独で十分なケースが多く、時間窓を加えるかはビジネス要件次第です。  
### より進んだ設計：Event Sourcing
削除・変更を「状態の更新」ではなく「イベントの追記」として扱う設計思想です[^3]。すべての変更履歴が自然に残り、任意の時点の状態を再現できます。ただし実装コストが高く、規模の小さな業務システムへの導入は慎重に判断すべきです。金融・医療・法務系など監査要件が厳しいシステムでは有力な選択肢となります。  
## 2. 名称変更時の過去データ保護
### 問題の本質
マスターテーブルの名称カラムを単純に `UPDATE` すると、過去の売上伝票を参照した際に「当時 A という名前で売ったのに、画面上では B になっている」という整合性の破壊が起きます。  
### 解決パターン
### ① スナップショット方式（名称コピー）
最もシンプルで、多くの ERP や EC サイトが採用している方法です。  
**設計：** トランザクションテーブル（売上伝票等）に `item_name` や `unit_price` のカラムを持たせ、確定時の値をそのままコピーして保存します。  
```sql
-- 受注テーブルの例
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    order_date  DATE,
    item_id     BIGINT,         -- マスタへの参照（現在の集計用）
    item_name   VARCHAR(200),   -- 注文時の名称をコピー（履歴保持用）
    unit_price  DECIMAL(10,2),  -- 注文時の単価をコピー
    ...
);
```
- **メリット：** マスタがどう変わろうと伝票作成時の状態が永久に固定されます。JOIN が減り集計クエリが高速になります  
- **デメリット：** ストレージ消費が増えますが、現代のDB設計では許容範囲内です  
**→ 実装コストと運用保守のバランスが最もよく、第一候補として推奨します。**  
### ② バージョニング方式（履歴保持）
「いつからいつまでこの名前だったか」をマスター自体で管理する方法です。SCD の **Type 2** に相当します[^4]。  
**設計：** マスタテーブルに `effective_date`（適用開始日）と `expiry_date`（適用終了日）を持たせ、名前を変えるときは既存レコードを UPDATE せず新レコードを INSERT します。  
```sql
-- 商品マスタ（バージョニング）
CREATE TABLE items (
    id              BIGINT PRIMARY KEY,
    item_code       VARCHAR(50),
    item_name       VARCHAR(200),
    effective_date  DATE NOT NULL,
    expiry_date     DATE,           -- NULL = 現在も有効
    ...
);

-- 売上日時点のマスタを結合するクエリ例
SELECT o.*, i.item_name
FROM orders o
JOIN items i
  ON i.item_code = o.item_code
 AND o.order_date BETWEEN i.effective_date AND COALESCE(i.expiry_date, '9999-12-31');
```
- **メリット：** 「過去の任意時点でのマスタ状態」を完全に再現できます  
- **デメリット：** SQL の結合条件が複雑になり、インデックス設計が難しくなります  
### SCD（Slowly Changing Dimensions）とは
データウェアハウス設計の権威 Ralph Kimball が提唱した、「緩やかに変化するマスターデータ」の分類です[^4]。  
| 型      | 概要                 | 対応パターン            |
| ------ | ------------------ | ----------------- |
| Type 1   | 上書き（履歴なし）          | 単純なUPDATE         |
| Type 2   | 新しい行を追加（完全履歴）      | バージョニング方式         |
| Type 3   | 前の値を別カラムに保持（限定的履歴）   | `prev_name` カラムなど   |
業務システムのマスター管理で「名称の変更履歴を残したい」という要件は、**Type 2** の採用を意味します。  
### ③ SQL Server Temporal Tables（システムバージョン管理テーブル）
SQL Server 2016 以降で利用可能なDB機能です。バージョニング方式の履歴管理をDB側が自動で行います[^5]。  
```sql
CREATE TABLE items (
    id         BIGINT PRIMARY KEY,
    item_name  VARCHAR(200),
    SysStartTime DATETIME2 GENERATED ALWAYS AS ROW START,
    SysEndTime   DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (SysStartTime, SysEndTime)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.items_history));

-- 特定時点のデータを取得
SELECT * FROM items
FOR SYSTEM_TIME AS OF '2025-01-01';
```
- **メリット：** アプリケーションコードに手を入れることなく、DBレベルで完全な変更履歴が自動管理されます  
- **デメリット：** SQL Server に依存します  
### ④ 別IDとして新規作成
名称変更がビジネス的に意味のある変化（組織改編による部署名変更など）を伴う場合に有効です。古いマスタを論理削除し、新しいIDで新レコードを作成します。  
サロゲートキー（ID）が変わらない限り、名称変更の影響は全データに及びます。過去と切り離したいなら「別のID（新しいレコード）」として扱うのがクリーンです。  
### パターン比較
| 要件                             | 推奨パターン            |
| ------------------------------ | ----------------- |
| 実装コストを最小化したい                   | スナップショット方式（名称コピー）   |
| 任意の時点に巻き戻して確認したい（SQL Server使用）   | Temporal Tables   |
| 任意の時点に巻き戻して確認したい（DB非依存）        | バージョニング方式         |
| 変更が組織・事業上の大きな区切り               | 別IDとして新規作成        |
## 3. 過去データの遡及修正
### エンタープライズにおける原則
**承認済み・集計済みの過去データを直接修正することは、エンタープライズの領域では原則禁止です。**  
理由は以下の2点です。  
1. **監査証跡の破壊**  
   - SOX法やISO監査基準では、取引記録の不変性が求められます  
   - 過去レコードの直接編集は事実上の改ざんとみなされる場合があります  
2. **整合性の連鎖崩壊**  
   - 月次集計・請求・在庫更新などの後続処理がすでに走っている場合、原レコードを修正すると関連するすべての集計値との整合性が取れなくなります  
   - マスターデータ（商品名・単価など）を遡及変更した場合、スナップショット方式を採用していない限り過去のすべての伝票表示に影響します  
### 正しい訂正の方法：訂正伝票（補正トランザクション）
過去の入力ミスや数量修正が必要な場合、**元レコードを上書きするのではなく、差異を新しいトランザクションとして起票**するのがエンタープライズの標準です。  
```plain text
【例：数量の入力ミス修正】

元の受注伝票：商品A × 10個 ＠100円 = 1,000円

訂正手順：
 ① 元の伝票はそのまま残す（変更しない）
 ② 訂正伝票を起票：商品A × -2個（訂正）= -200円
 ③ 正味：8個 = 800円

→ 元の伝票と訂正伝票の両方が監査ログに残る
```
この手法により「なぜ数量が変わったのか」という経緯がすべてトレース可能になります。  
### 許容される「訂正」の範囲
| 操作              | 可否       | 理由             |
| --------------- | -------- | -------------- |
| まだ承認前の伝票の修正     | ○ 可      | 後続処理が走っていないため   |
| 承認済み伝票への訂正伝票の起票   | ○ 可      | 元データを変えず差異を記録   |
| 承認済み伝票の直接上書き    | △ 要承認フロー   | 監査ログとセットで厳格に管理   |
| 集計済みデータの遡及変更    | × 原則禁止   | 整合性が崩壊する       |
| マスターデータの遡及変更    | × 原則禁止   | 過去の全集計に影響する    |
## 4. 設計指針の統合
**削除系**  
```plain text
ユーザーが「削除」を要求
        ↓
トランザクションデータからの参照チェック
        ↓
  参照あり ──→ 物理削除をブロック → 廃止（論理削除）へ
        ↓
  参照なし かつ Draft 状態 ──→ 物理削除を許可
```
**名称変更系**  
```plain text
ユーザーが名称変更を要求
        ↓
スナップショット方式を採用している場合
  → マスタを自由に UPDATE（過去データは伝票側に固定済み）
        ↓
スナップショット方式を採用していない場合
  → Temporal Tables またはバージョニング方式を検討
```
## まとめ：迷ったらこのパターンを採用する
**削除については「論理削除＋削除禁止の組み合わせ」**  
- 論理削除の実装は `is_deleted` フラグ・`Status` enum どちらでもよい。「複数の意味ある状態がビジネス上存在するか」で選ぶ  
- いずれの場合も `deleted_at` と `deleted_by` の audit 列を合わせて持たせる  
- 他人との衝突が起きるユニーク制約には**部分インデックス**で対応する  
- 同一エンティティを復活させたい場合は**Reactivate**、ビジネス的に別物なら**新規作成**  
- トランザクションデータからの参照がある場合は物理削除をブロック  
- 台帳画面には削除ボタンではなく「無効化」ボタンを設置  
**名称変更については「スナップショット方式（名称コピー）」**  
- 伝票確定時にマスタ名称をトランザクションテーブルへコピー  
- マスタ側は自由に変更・無効化してよい  
- SQL Server を使用している場合は Temporal Tables も有力な選択肢  
**過去データの修正については「訂正伝票の起票」**  
- 承認済みの過去データを直接書き換えることは原則禁止  
- 差異を新しいトランザクションとして起票し、監査証跡を残す  
## 参考リソース
### EF Core（論理削除の実装）
| 内容                                     | URL                                                        |
| -------------------------------------- | ---------------------------------------------------------- |
| EF Core Global Query Filters（公式ドキュメント）   | https://learn.microsoft.com/en-us/ef/core/querying/filters   |
### DDD・設計思想
| 内容                                                   | URL                                            |
| ---------------------------------------------------- | ---------------------------------------------- |
| Martin Fowler - Analysis Patterns（Temporal Patterns）   | https://martinfowler.com/books/ap.html         |
| DMBOK（データマネジメント知識体系ガイド）                              | https://www.dama.org/cpages/body-of-knowledge   |
| Soft Delete Anti-pattern（Depesz）                     | https://www.depesz.com/2010/01/12/soft-delete/   |
### SCD（緩やかに変化するディメンション）
| 内容                      | URL                                                                                                                                                              |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Kimball Group - SCD の解説   | https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/slowly-changing-dimension-type-1/   |
### SQL Server Temporal Tables
| 内容                                        | URL                                                                                                                     |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Temporal Tables 公式ドキュメント（Microsoft Learn）   | https://learn.microsoft.com/ja-jp/sql/relational-databases/tables/temporal-tables                                       |
| Temporal Tables 入門（Microsoft Learn）       | https://learn.microsoft.com/ja-jp/sql/relational-databases/tables/getting-started-with-system-versioned-temporal-tables   |
### Event Sourcing
| 内容                                                 | URL                                                                          |
| -------------------------------------------------- | ---------------------------------------------------------------------------- |
| Event Sourcing パターン（Microsoft Azure Architectures）   | https://learn.microsoft.com/ja-jp/azure/architecture/patterns/event-sourcing   |
---
[^1]: EF Core Global Query Filters — https://learn.microsoft.com/en-us/ef/core/querying/filters  
[^2]: Eric Evans『Domain-Driven Design』（2003）  
[^3]: Event Sourcing パターン — https://learn.microsoft.com/ja-jp/azure/architecture/patterns/event-sourcing  
[^4]: Kimball Group - Slowly Changing Dimensions — https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/slowly-changing-dimension-type-1/  
[^5]: Microsoft Learn - Temporal Tables — https://learn.microsoft.com/ja-jp/sql/relational-databases/tables/temporal-tables  
