---
title: "dotnet ユーザーシークレット"
type: "Tech"
description: ".NET の User Secrets を用いた機密情報管理のまとめ。プロジェクトファイル（.csproj）への ID 付与から CLI での値設定、コード内での IOptions パターンによる読み出し方まで。Git への誤コミットを防ぎ、セキュアなローカル開発環境を構築するための手順書。

  • https://zenn.dev/rendya/articles/dotnet-user-secrets-note
  • https://claude.ai/chat/52c70a60-1ca6-4459-a21e-c449c5d68dca"
tags: [".NET","Security","C#"]
date: "2026-03-20T00:00:00"
---

 メモ  
# 概要
機密情報をプロジェクト外に保存する仕組み。Git に含まれない構造的な安全性が目的。暗号化はされない。  
## 保存場所
| OS            | パス                                                  |
| ------------- | --------------------------------------------------- |
| Windows       | `%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json`   |
| macOS / Linux   | `~/.microsoft/usersecrets/<id>/secrets.json`        |
`<id>` は `.csproj` に記録される GUID（`UserSecretsId`）。  
```xml
<!-- .csproj に自動追加される -->
<PropertyGroup>
  <UserSecretsId>xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx</UserSecretsId>
</PropertyGroup>
```
この GUID によって複数プロジェクトのシークレットが混在しないよう管理される。意図的に同じ GUID を設定すれば複数プロジェクトで共有できる。  
## secrets.json の構造
`appsettings.json` と同じ JSON 形式。コロン区切りのキーとネスト構造は等価。  
```json
// フラット形式
{
  "Api:Key": "my-secret-key",
  "ConnectionStrings:DefaultConnection": "Server=...;Password=secret"
}

// ネスト形式（どちらでも同じ結果）
{
  "Api": {
    "Key": "my-secret-key"
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=...;Password=secret"
  }
}
```
## 設定プロバイダーとしての位置づけ
`IConfiguration` は複数のプロバイダーを重ね合わせる構造で、後から追加されたプロバイダーが優先される。  
`WebApplication.CreateBuilder()` でのデフォルトの読み込み順（優先度 低 → 高）：  
1. `appsettings.json`  
2. `appsettings.{Environment}.json`  
3. ユーザーシークレット（`Development` 環境のみ）  
4. 環境変数  
5. コマンドライン引数  
ユーザーシークレットは `appsettings.json` を上書きするが、環境変数には上書きされる。  
## セットアップと基本コマンド
`.csproj` があるディレクトリで実行するのが基本。別の場所から実行する場合は `--project` オプションで指定。`set` や `list` などすべてのコマンドで同様に使える。  
```bash
# プロジェクトディレクトリ内で実行
cd MyProject
dotnet user-secrets init

# 別の場所から実行する場合
dotnet user-secrets init --project ./MyProject
```
### 追加・更新
`set` が追加と更新を兼ねる。既存キーに実行すると上書きされる。  
```bash
dotnet user-secrets set "Api:Key" "my-secret-key"
dotnet user-secrets set "Api:BaseUrl" "https://api.example.com"
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=...;Password=secret"
```
### 一覧表示
```bash
dotnet user-secrets list
```
出力例：  
```plain text
Api:BaseUrl = https://api.example.com
Api:Key = my-secret-key
ConnectionStrings:DefaultConnection = Server=...;Password=secret
```
### 削除
```bash
dotnet user-secrets remove "Api:Key"
dotnet user-secrets clear  # 全削除（ファイル自体は残る）
```
### JSON で一括登録
JSON が不正な場合は登録時点でエラーになり何も登録されない。  
```json
// secrets.json
{
  "Api:Key": "my-secret-key",
  "Api:BaseUrl": "https://api.example.com",
  "ConnectionStrings:DefaultConnection": "Server=...;Password=secret"
}
```
```bash
# macOS / Linux / Git Bash
cat secrets.json | dotnet user-secrets set

# Windows コマンドプロンプト
type secrets.json | dotnet user-secrets set
```
```powershell
# Windows PowerShell（-Raw で1つの文字列として渡す必要がある）
Get-Content secrets.json -Raw | dotnet user-secrets set

# cat は Get-Content のエイリアスのため -Raw が必要
cat secrets.json -Raw | dotnet user-secrets set
```
## アプリの種類別：読み込みとアクセス方法
### ASP.NET Core の場合
`ASPNETCORE_ENVIRONMENT=Development` のとき `IConfiguration` に自動統合される。追加設定不要。  
```c#
var builder = WebApplication.CreateBuilder(args);
// この時点で builder.Configuration にユーザーシークレットが含まれている
```
**直接取得：**  
```c#
var apiKey = builder.Configuration["Api:Key"];
var connStr = builder.Configuration.GetConnectionString("DefaultConnection");
```
**Options パターン経由：**  
`GetSection("Api")` は `Api:` プレフィックスを持つキーをまとめて取得し、プロパティ名に対応させてバインドする。  
```bash
dotnet user-secrets set "Api:Key" "my-secret-key"
dotnet user-secrets set "Api:BaseUrl" "https://api.example.com"
```
```c#
public class ApiSettings
{
    public string Key { get; set; } = string.Empty;
    public string BaseUrl { get; set; } = string.Empty;
}

// Program.cs で登録
builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("Api"));

// DI 経由で利用
public class MyService(IOptions<ApiSettings> options)
{
    private readonly ApiSettings _settings = options.Value;

    public void DoSomething()
    {
        var key = _settings.Key;      // "my-secret-key"
        var url = _settings.BaseUrl;  // "https://api.example.com"
    }
}
```
### コンソールアプリ・テストプロジェクトの場合
自動統合されないため `ConfigurationBuilder` で明示的に読み込む。型引数にはそのプロジェクト内の任意のクラスを指定する（指定したクラスが属するアセンブリの `UserSecretsId` が使われる）。  
```c#
var config = new ConfigurationBuilder()
    .AddUserSecrets<Program>()  // テストプロジェクトなら任意のクラスでよい
    .Build();
```
**直接取得：**  
```c#
var apiKey = config["Api:Key"];
var connStr = config.GetConnectionString("DefaultConnection");
```
**キーが未設定の場合に例外を出す：**  
```c#
var apiKey = config["Api:Key"]
    ?? throw new InvalidOperationException("User Secrets に Api:Key が設定されていません。");
```
`optional` 引数でキーが見つからない場合の挙動を制御できる。デフォルトは .NET 6.0 以降 `false`（未設定で例外）。.NET 5 以前は `true` がデフォルトだった。  
```c#
.AddUserSecrets<Program>(optional: true)   // UserSecretsId が未設定でも例外を出さない
.AddUserSecrets<Program>(optional: false)  // UserSecretsId が未設定なら例外を出す（デフォルト）
```
**Options パターン経由：**  
`ServiceCollection` を手動で構築することで ASP.NET Core と同じ仕組みを利用できる。  
```c#
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;

var config = new ConfigurationBuilder()
    .AddUserSecrets<Program>()
    .Build();

var services = new ServiceCollection();
services.Configure<ApiSettings>(config.GetSection("Api"));

var provider = services.BuildServiceProvider();
var settings = provider.GetRequiredService<IOptions<ApiSettings>>().Value;

Console.WriteLine(settings.Key);     // "my-secret-key"
Console.WriteLine(settings.BaseUrl); // "https://api.example.com"
```

