---
title: "ASP.NET Controller 設計まとめ"
type: "Tech"
description: "ベストプラクティス / アンチパターン / エラーハンドリング

  • https://chatgpt.com/c/696ee7b9-87d4-8324-b20a-7e514ff6fd54"
tags: ["C#","ASP.Net"]
date: "2026-01-25T00:00:00"
---


## 1. コントローラーの責務とは？
> Controller = HTTPとアプリケーション層の境界アダプタ  
### やるべきこと（責務）
- ルーティング  
- モデルバインディング（Body / Query / Route）  
- 認証・認可（\[Authorize\]など）  
- アプリ層（Service / UseCase）を呼ぶ  
- 結果をHTTPレスポンスへ変換  
- ステータスコード決定（200 / 404 / 400など）  
### やってはいけないこと
- ビジネスロジックを書く  
- DB操作を書く  
- 例外を握りつぶす  
- if文だらけの業務分岐を書く  
- 仕様の中核ルールを書く  
---
## 2. コントローラーで try-catch してレスポンス変換するのはあり？
### 基本方針
- ❌ 各コントローラーで try-catch しまくる → アンチパターン  
- ⭕ 例外はミドルウェアで一元処理  
- ⭕ 業務上の失敗は Result型 で返す  
### 悪い例（アンチパターン）
```c#
try {
    service.Execute();
    return Ok();
}
catch (NotFoundException) {
    return NotFound();
}
catch (ValidationException) {
    return BadRequest();
}
```
→ コントローラーが肥大化し、責務違反  
---
## 3. Result型 = ユニオン型？
はい、実質的にそう。  
```c#
Result<User> =
    Success(User)
  | NotFound
  | ValidationError
  | Conflict
```
これは「代数的データ型（Union型）」と同じ構造。  
---
## 4. なぜ Result型が重要か？
### 問題点（従来）
- nullを返す  
- boolで返す  
- 例外で制御する  
→ 型から「失敗しうる」ことが分からない  
### 解決策
Result型を使うことで：  
```c#
Result<User> FindUser(id)
```
→ 呼び出し側は「失敗ケースを必ず考慮する設計」になる  
---
## 5. ビジネスエラーとシステムエラーの区別
種類                 扱い  
---
ユーザー入力エラー   Resultで返す  
データ未存在         Resultで返す  
業務ルール違反       Resultで返す  
NullReference        例外  
DB接続エラー         例外  
バグ                 例外  
> Result = 想定内の分岐\  
> Exception = 想定外の異常  
---
## 6. コントローラーにswitchを書くのがつらい問題
### 解決法：共通マッピング関数を作る
```c#
public static IResult ToHttp<T>(this Result<T> result) =>
    result switch
    {
        Success<T> s => Results.Ok(s.Value),
        NotFound => Results.NotFound(),
        ValidationError v => Results.BadRequest(v.Errors),
        _ => Results.Problem()
    };
```
Controller側はこう書ける：  
```c#
return (await useCase.Execute()).ToHttp();
```
---
## 7. バリデーションエラーは例外？Result？
### 現代的ベストプラクティス
- バリデーションエラー = 業務上よくある分岐  
- よって「例外ではなくResultで返す」が自然  
### 悪い例（制御に例外を使っている）
```c#
throw new ValidationException(...);
```
### 良い例
```c#
return Result.Invalid(errors);
```
---
## 8. MVC全盛期の実情（歴史的背景）
昔は以下が普通だった：  
```c#
throw new Exception("User not found");
```
```c#
catch(Exception ex) {
    ViewBag.Error = ex.Message;
    return View();
}
```
- 検索0件 → 例外  
- バリデーション失敗 → 例外  
- 重複登録 → 例外  
理由： - Result型文化がなかった - C#にUnion型がない - 「失敗 =  
例外」という直感  
→ 今見るとかなりカオス  
---
## 9. ベストプラクティスまとめ
### 理想構成
```plain text
Controller
   ↓
UseCase / Service → Result<T>
   ↓
ResultMapper（共通変換）
   ↓
HTTPレスポンス
```
例外はここだけで拾う：  
```plain text
ExceptionMiddleware → 500 ProblemDetails
```
---
## 10. よくあるアンチパターン一覧
アンチパターン                 問題  
---
Controllerにビジネスロジック   保守不能になる  
すべて例外で制御               可読性崩壊  
nullで失敗表現                 型安全性ゼロ  
if文だらけController           設計崩壊のサイン  
try-catchだらけ                横断関心事を分離できていない  
---
## 11. 結論
> 良いControllerとは「薄いController」である  
- ロジックを持たない  
- 分岐を持たない  
- 変換と委譲だけをする  
- 型（Result）に責務を語らせる  
これが現代的なASP.NET設計の到達点。  
