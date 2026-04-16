---
title: "N+1問題とは？"
type: "Tech"
description: "・https://gemini.google.com/app/1c0297329920dc09
・https://claude.ai/chat/1fe8c3f3-4ee7-469f-a2ea-10afbcc082cf"
date: "2026-04-15T00:00:00"
---

## N+1問題とは？
**「1回のクエリで取得したN件のレコードそれぞれに対して、追加クエリをN回発行してしまう」** パフォーマンス問題のこと。  
合計クエリ数が **1 + N** 回になるのが問題の本質。  
一言で言えば、**「効率の悪いループ処理」がデータベースに対して行われている状態**を指します。  

## 技術的な発生メカニズム
ブログの「記事（Posts）」と、その「投稿者（Users）」を表示するシステムを例に、何が起きているかを見てみましょう。  
```c#
// 1. まず全記事を取得する（これが「1」のクエリ）
var posts = db.Posts.ToList(); 

foreach (var post in posts) {
    // 2. 各記事ごとに、投稿者の名前を取得する（これが「N」のクエリ）
    // ループの中で毎回 SELECT * FROM Users WHERE id = ... が走る
    Console.WriteLine(post.Title + " by " + post.User.Name); 
}
```
### 発行されるSQL
1. `SELECT * FROM Posts;` （ここで記事が100件取れたとする）  
2. `SELECT * FROM Users WHERE id = 1;`  
3. `SELECT * FROM Users WHERE id = 2;`  
   ...  
4. `SELECT * FROM Users WHERE id = 100;`  
合計で **1 + 100 = 101回** のクエリが投げられます。これがN+1問題の正体です。  

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



