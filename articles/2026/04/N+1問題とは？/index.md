---
title: "N+1問題とは？"
type: "Tech"
description: "・https://gemini.google.com/app/1c0297329920dc09
・https://claude.ai/chat/1fe8c3f3-4ee7-469f-a2ea-10afbcc082cf"
tags: ["C#","DB","SQL"]
date: "2026-04-15T00:00:00"
---

## N+1問題とは？
**「1回のクエリで取得したN件のレコードそれぞれに対して、追加クエリをN回発行してしまう」** パフォーマンス問題のこと。  
合計クエリ数が **1 + N** 回になるのが問題の本質。  

## 技術的な発生メカニズム
ユーザー一覧ページで、各ユーザーの投稿数を表示するケース。  
❌ N+1が発生するコード（C# / EF Core）  
```c#
// クエリ①：1回
var users = await context.Users.ToListAsync(); 

foreach (var user in users) // Nユーザー分ループ
{
    // クエリ②〜N+1：毎回DBアクセスが走る
    var postCount = await context.Posts
        .CountAsync(p => p.UserId == user.Id);
    Console.WriteLine($"{user.Name}: {postCount}件");
}
```
発行されるSQL（Nが100なら101回）  
```sql
SELECT * FROM Users;                                   -- 1回
SELECT COUNT(*) FROM Posts WHERE UserId = 1;           -- ↓
SELECT COUNT(*) FROM Posts WHERE UserId = 2;           -- N回
SELECT COUNT(*) FROM Posts WHERE UserId = 3;           -- ↓
```

## 解決策
### 解決策①：Eager Loading（`Include`）
EF Coreで最もシンプルな解決策。関連データを最初から一緒に取得する。  
```c#
var users = await context.Users
    .Include(u => u.Posts) // JOINで一括取得
    .ToListAsync();

foreach (var user in users)
{
    Console.WriteLine($"{user.Name}: {user.Posts.Count}件");
    // DBアクセスなし。既にメモリ上にある
}
```
**発行されるSQL（1回のみ）**  
```sql
SELECT u.*, p.*
FROM Users u
LEFT JOIN Posts p ON p.UserId = u.Id;
```
**⚠️ 注意点：** 投稿数だけ欲しいのに全投稿データを引いてくるので、件数が多い場合はメモリの無駄遣いになる。  
### 解決策②：Projection（必要なデータだけSELECT）
集計だけ必要な場合はこちらが最適。  
```c#
var result = await context.Users
    .Select(u => new
    {
        u.Name,
        PostCount = u.Posts.Count() // サブクエリとして最適化される
    })
    .ToListAsync();
```
**発行されるSQL（1回のみ）**  
```sql
SELECT u.Name, (SELECT COUNT(*) FROM Posts WHERE UserId = u.Id) AS PostCount
FROM Users u;
```
### 解決策③：明示的なJOIN + GroupBy
より複雑な集計やDapperなどのMicro-ORMを使う場合。  
```c#
// EF Core
var result = await context.Users
    .GroupJoin(
        context.Posts,
        u => u.Id,
        p => p.UserId,
        (u, posts) => new { u.Name, PostCount = posts.Count() }
    )
    .ToListAsync();
```
```sql
-- 生SQL（Dapper等で直接書く場合）
SELECT u.Name, COUNT(p.Id) AS PostCount
FROM Users u
LEFT JOIN Posts p ON p.UserId = u.Id
GROUP BY u.Id, u.Name;
```
### 解決策④：バッチ取得（IN句）
EF Coreではなく手動でIDを集めてまとめて取得するパターン。  
```c#
var users = await context.Users.ToListAsync();
var userIds = users.Select(u => u.Id).ToList();

// IN句で一括取得
var postCounts = await context.Posts
    .Where(p => userIds.Contains(p.UserId))
    .GroupBy(p => p.UserId)
    .Select(g => new { UserId = g.Key, Count = g.Count() })
    .ToDictionaryAsync(x => x.UserId, x => x.Count);

foreach (var user in users)
{
    var count = postCounts.GetValueOrDefault(user.Id, 0);
    Console.WriteLine($"{user.Name}: {count}件");
}
```

## 各解決策の使い分け
| 状況             | 推奨手法                     |
| -------------- | ------------------------ |
| 関連エンティティを丸ごと使う   | `Include`（Eager Loading）   |
| 集計値だけ必要        | `Select` Projection      |
| 複雑な集計・Dapper使用   | 生SQL（JOIN + GROUP BY）    |
| 既存コードを最小限変更したい   | バッチ取得（IN句）               |

## なぜ「1+N」じゃなく「N+1」と呼ぶのか
> **「問題の深刻度はNがどれだけ大きいかで決まる」**  
という視点から、**N（問題の主体）を前に置いて強調**しているのが理由。  
`クエリ発行順序（時系列）: 1 + N  ← 処理の流れとしてはこっち  
問題の重大度・命名観点: N + 1  ← Nが大きいほど深刻なのでNを前に`  
1回の親クエリはおまけで、**N回のループクエリこそが「問題の正体」** だという命名の意図がある。英語圏でも同様に "N+1 problem" と呼ばれており、慣習として定着している。  

## なぜORMでN+1が起きやすいのか
根本原因は**「オブジェクト思考」と「DB思考」のミスマッチ**。  
```c#
// 開発者の頭の中はオブジェクト操作
foreach (var user in users)
{
    user.Posts // ← 「プロパティを参照してるだけ」のつもり
}

// 実際にはDBアクセスが走っている（Lazy Loading）
```
生SQLで書く人は最初から「どうSELECTするか」を考える。ORMはその思考を**オブジェクト操作で隠蔽**するから、裏でN回クエリが走っていても気づかない。  

### ORM以外でN+1が起きるケース
ただし、N+1はORM固有の問題ではない。**「ループの中でI/Oを呼ぶ」構造**ならどこでも起きる。  

**GraphQL Resolver（有名な発生源）**  
```javascript
// 各userフィールドのresolverが独立して呼ばれる
const resolvers = {
  User: {
    posts: (user) => db.query(`SELECT * FROM posts WHERE user_id = ${user.id}`)
    // userの数だけこのresolverが呼ばれる → N+1
  }
}
```
GraphQLはN+1があまりにも頻発するので**DataLoader**というバッチ解決の専用ライブラリが生まれた。  


**マイクロサービス間API呼び出し**  
```c#
var orders = await orderService.GetAllAsync(); // 1回

foreach (var order in orders)
{
    // 別サービスへHTTPリクエスト → N回
    var user = await userService.GetByIdAsync(order.UserId);
}
```
DBではなくHTTPになっただけで構造は同じ。しかもネットワークレイテンシが乗るので**DBのN+1より深刻になりうる**  

### なぜORMでN+1が起きやすいのかまとめ
|            | ORM                          | 生SQL                 |
| ---------- | ---------------------------- | -------------------- |
| N+1の発生しやすさ   | **高い**（Lazy Loadingで無自覚に）    | **低い**（JOIN前提で考えるから）   |
| 気づきにくさ     | **高い**（抽象化で隠れる）              | **低い**（クエリが見えている）    |
| ORM以外での発生   | GraphQL / マイクロサービス / アプリ層ループ   |                      |
**本質はORM問題というより「抽象化が隠すコスト問題」**。生SQLを書く人がN+1を考えたことすらないのは、抽象化レイヤーがなくてコストが常に見えているから。ORMはその見通しを奪う。  

## Eager Loading とは
**「必要になるかもしれない関連データを、最初のクエリと同時に先読みしておく」** 戦略。  
対になる概念と並べると理解しやすい。  
| 戦略                   | タイミング      | 挙動                 |
| -------------------- | ---------- | ------------------ |
| **Eager Loading**    | 最初のクエリ時    | 関連データを即座にJOINで取得   |
| **Lazy Loading**     | プロパティアクセス時   | アクセスされた瞬間にDBへ追加クエリ   |
| **Explicit Loading**   | 任意のタイミング   | 明示的に`Load()`を呼んで取得   |
Lazy Loadingが**N+1の温床**で、Eager Loadingがその**標準的な解決策**という位置づけ。  

## EF Core の `Include` とは
Eager Loadingを実現するEF CoreのAPI。**「このナビゲーションプロパティも一緒に取ってきて」** という指示。  
```c#
// エンティティ定義
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<Post> Posts { get; set; } // ナビゲーションプロパティ
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public int UserId { get; set; }
    public User User { get; set; } // ナビゲーションプロパティ（逆方向）
}
```
```c#
// Includeなし → Lazy Loadingでアクセス時にN+1が走る可能性
var users = await context.Users.ToListAsync();

// Includeあり → JOINで一括取得
var users = await context.Users
    .Include(u => u.Posts)
    .ToListAsync();
```
発行されるSQL  
```sql
SELECT u.*, p.*
FROM Users u
LEFT JOIN Posts p ON p.UserId = u.Id
```

## ThenInclude：ネストした関連も取得
関連の関連まで取りたい場合。  
```c#
// User → Posts → Comments まで一括取得
var users = await context.Users
    .Include(u => u.Posts)
        .ThenInclude(p => p.Comments)
    .ToListAsync();
```
```sql
SELECT u.*, p.*, c.*
FROM Users u
LEFT JOIN Posts p ON p.UserId = u.Id
LEFT JOIN Comments c ON c.PostId = p.Id
```

複数の関連を同時取得  
```c#
// Postsとprofileを両方取る
var users = await context.Users
    .Include(u => u.Posts)
    .Include(u => u.Profile)
    .ToListAsync();
```

## Include の落とし穴
### ① 不要なデータまで全件取得する
```c#
// 投稿数だけ欲しいのに全投稿データを引いてくる
var users = await context.Users
    .Include(u => u.Posts) // Postsの全カラム・全件がメモリに乗る
    .ToListAsync();

var count = user.Posts.Count; // これだけのためにIncludeは重すぎる
```
集計・絞り込みが目的なら`Select` Projectionの方が適切。  
```c#
// こっちが正解
var users = await context.Users
    .Select(u => new { u.Name, PostCount = u.Posts.Count() })
    .ToListAsync();
```

### ② Includeにフィルターをかけたい場合
EF Core 5.0以降は**Filtered Include**が使える。  
```c#
// 公開済みの投稿だけInclude
var users = await context.Users
    .Include(u => u.Posts.Where(p => p.IsPublished))
    .ToListAsync();
```

## まとめ
- **Eager Loading** = 関連データを最初から先読みする戦略  
- **Include** = EF CoreでEager Loadingを実現するAPI  
- 「関連エンティティを丸ごと使う」場面で有効  
- **集計・絞り込みだけが目的なら****`Select`** **Projectionの方が効率的**  
`Include`はN+1の解決策として紹介されることが多いけど、**「全データが必要なケースに使う道具」** という認識で使い分けるのが正しい。  