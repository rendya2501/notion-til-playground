---
title: "アノテーション vs アトリビュート：JavaとC#の用語比較"
type: "Tech"
description: "Javaはアノテーション、C#はアトリビュート

  • https://claude.ai/chat/2d7691f5-f95c-489a-ad64-6b42908ee1f8"
tags: ["C#","Java"]
date: "2026-03-02T00:00:00"
---


JavaとC#で「似たような仕組みなのに呼び方が違う」と混乱しやすい **アノテーション（Annotation）** と **アトリビュート（Attribute）** を整理します。  

---
## ① それぞれの正式名称と定義
|        | Java                    | C#                     |
| ------ | ----------------------- | ---------------------- |
| 正式な呼び名   | **アノテーション**（Annotation）   | **アトリビュート**（Attribute）   |
| 語源     | 「注釈・注記」                 | 「属性・特性」                |
| 書き方    | `@名前`                   | `[名前]`                 |
| 使う記号   | `@`（アットマーク）             | `[ ]`（角かっこ）            |
どちらも「コードの要素に対してメタデータ（付加情報）を付与する仕組み」で、**役割は全く同じ**です。呼び方と書き方が言語によって異なるだけです。  
---
## ② 何のために使うのか
大きく3つの用途があります。  
### フレームワークへの指示
「このメソッドはトランザクション管理してほしい」「このエンドポイントは認証が必要」など、フレームワークに処理を委ねるための合図として使います。  
```java
// Java
@Transactional
public void transferMoney() { ... }
```
```c#
// C#
[Authorize(Roles = "Admin")]
public IActionResult Dashboard() { ... }
```
### バリデーション（入力検証）
フィールドの値に制約をつけます。「nullは禁止」「最大100文字まで」など。  
```java
// Java
@NotNull
@Size(max = 50)
private String name;
```
```c#
// C#
[Required]
[StringLength(50)]
public string Name { get; set; }
```
### コンパイラへのヒント
「このメソッドはオーバーライドだ」「このコードは非推奨だ」などをコンパイラに伝えます。  
```java
// Java
@Override
public String toString() { ... }

@Deprecated
public void oldMethod() { ... }
```
```c#
// C#
// overrideはキーワードとして存在するため属性は不要
public override string ToString() { ... }

[Obsolete]
public void OldMethod() { ... }
```
---
## ③ 書き方の比較
### 基本形
```java
// Java：@ + アノテーション名
@Transactional
public void save() { ... }
```
```c#
// C#：[ ] の中にアトリビュート名
[Transactional]
public void Save() { ... }
```
### パラメータ（引数）を渡す場合
```java
// Java：丸かっこ () でパラメータを渡す
@Transactional(readOnly = true)
public List<User> findAll() { ... }

@Column(name = "user_name", nullable = false, length = 50)
private String username;
```
```c#
// C#：角かっこの中で丸かっこを使う
[Column("user_name")]
public string Username { get; set; }

[StringLength(50, MinimumLength = 3)]
public string Name { get; set; }
```
---
## ④ 詳細比較テーブル
### 基本情報
| 比較項目     | ☕ Java：アノテーション         | # C#：アトリビュート                       |
| -------- | ---------------------- | ---------------------------------- |
| 正式な呼び名   | アノテーション（Annotation）    | アトリビュート（Attribute）                 |
| 書き方      | `@名前` または `@名前(引数)`    | `[名前]` または `[名前(引数)]`              |
| 使う記号     | `@`（アットマーク）            | `[ ]`（角かっこ）                        |
| 定義の継承元   | 特殊な `@interface` 形式    | `System.Attribute` クラスを継承          |
| 主な利用場面   | Spring / Jakarta EE など   | ASP.NET Core / Entity Framework など   |
| 語源のニュアンス   | 「注釈」＝コードにメモを添えるイメージ    | 「属性」＝その要素の性質を定義するイメージ              |
### 代表的な用途の対応表
| 用途          | ☕ Java                      | # C#                             |
| ----------- | --------------------------- | -------------------------------- |
| オーバーライドの明示   | `@Override`                 | `override` キーワード（属性不要）           |
| 非推奨の宣言      | `@Deprecated`               | `[Obsolete]`                     |
| null禁止・必須   | `@NotNull`                  | `[Required]`                     |
| 文字数制限       | `@Size(max = 50)`           | `[StringLength(50)]`             |
| トランザクション管理   | `@Transactional`（Spring）    | 標準属性なし（`TransactionScope` を使用）   |
| 認証・認可       | `@Secured`（Spring Security）   | `[Authorize]`（ASP.NET Core）      |
| DBカラムのマッピング   | `@Column(name = "...")`     | `[Column("...")]`                |
| テストメソッドの宣言   | `@Test`（JUnit）              | `[Test]`（NUnit）/ `[Fact]`（xUnit）   |
---
## ⑤ 混乱しないための覚え方
- **Java → アノテーション（****`@`****）**  
  `@`（アットマーク）の "A" と "A"nnotation の "A" で揃えて覚える。  
- **C# → アトリビュート（****`[ ]`****）**  
  - [ ] `[ ]`（角かっこ）= 属性を入れる「箱」= アトリビュート。  
- **言い間違えても大丈夫**  
  役割は完全に同じなので、エンジニア同士なら「あの `@` のやつ / `[ ]` のやつ」と即座に通じます。  
---
## まとめ
> **役割は同じ・名前と記号だけが違う。**  
> Javaでは `@` を使い「アノテーション」、C#では `[ ]` を使い「アトリビュート（属性）」と呼ぶ。  
どちらも「コードの要素にメタデータを付与し、フレームワークやコンパイラへ指示を出す」という目的で使われます。一方の言語を知っていれば、もう一方の概念はほぼそのまま理解できます。  


