---
title: "Docker × ASP.NET Core 設定管理：.env と環境変数の上書き"
type: "Tech"
description: "docker-compose.yml のシークレット直書きを .env に切り出し、ASP.NET Core の環境変数上書きの仕組みを使って appsettings.json を触らずに設定を注入する方法

  • https://claude.ai/chat/c26e5815-17ae-4b0e-8469-d16a92aac1aa"
tags: [".NET","Docker"]
date: "2026-03-21T00:00:00"
---

## 背景：なぜこの対応が必要だったか
`docker-compose.yml` にパスワードを直書きしていたところ、GitGuardian（GitHub連携のシークレット検出ツール）に検知されて警告が発生した。テスト環境の値でも、パブリックリポジトリに上げると警告が出る。  
## 1. `.env` と `docker-compose.yml` の連携
### 仕組み
Docker Compose は**プロジェクトルートに** **`.env`** **ファイルがあると自動的に読み込む**。  
明示的な設定は不要で、`docker-compose up` を実行するだけで有効になる。  
### 設定例
**`.env`**  
```plain text
POSTGRES_DB=clean-architecture
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```
**`docker-compose.yml`**（`${変数名}` で参照）  
```yaml
postgres:
  environment:
    - POSTGRES_DB=${POSTGRES_DB}
    - POSTGRES_USER=${POSTGRES_USER}
    - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
```
### ⚠️ `.gitignore` への追加は最重要
`.env` を作成したら**即座に** `.gitignore` に追加すること。`git add` した瞬間にシークレットがリポジトリに記録され、その後削除しても Git の履歴には残り続ける。後から気づいても完全な消去は困難なため、ファイル作成と同時にやる習慣をつける。  
```plain text
.env
```
### `.env.example` をコミット用に用意する
実際の値は書かず、キー名だけ記載したファイルをコミットしておくと、リポジトリを見た人が「何の変数が必要か」を把握できる。  
```plain text
POSTGRES_DB=
POSTGRES_USER=
POSTGRES_PASSWORD=
```
## 2. docker-compose → ASP.NET Core 設定の上書き
### ASP.NET Core の設定優先順位
ASP.NET Core の設定システムは複数のソースを**優先度順に重ね合わせる**仕組みになっている。  
```plain text
低 ←────────────────────────────────────── 高
appsettings.json < appsettings.{Environment}.json < 環境変数
```
**環境変数が最も強く、appsettings.json を上書きする。**  
`docker-compose.yml` の `environment:` セクションに書いた値は、コンテナ起動時に OS の環境変数としてプロセスに渡される。ASP.NET Core はその環境変数を読むため、**docker-compose の** **`environment:`** **は優先順位の「環境変数」に該当する**。  
### docker-compose から web-api に接続文字列を注入する
```yaml
web-api:
  environment:
    - ConnectionStrings__DefaultConnection=Host=postgres;Port=5432;Database=${POSTGRES_DB};Username=${POSTGRES_USER};Password=${POSTGRES_PASSWORD}
```
### ネスト構造のキー名ルール
ASP.NET Core は環境変数名の `__`（アンダースコア2つ）を `:` と同等に解釈する。  
| 環境変数名                                  | appsettings.json のキー                  |
| -------------------------------------- | ------------------------------------- |
| `ConnectionStrings__DefaultConnection`   | `ConnectionStrings:DefaultConnection`   |
| `Jwt__Secret`                          | `Jwt:Secret`                          |
この変換ルールにより、`appsettings.json` を直接書き換えずに値を差し替えられる。  
## 3. GitGuardian 警告への対処まとめ
| 対処                        | 効果                                |
| ------------------------- | --------------------------------- |
| パスワードを `.env` に移動         | `docker-compose.yml` からシークレットを排除   |
| `.env` を `.gitignore` に追加   | Git に `.env` が含まれなくなる             |
| `web-api` に環境変数で接続文字列を注入   | `appsettings.json` にもパスワードを書かずに済む   |
| `.env.example` をコミット      | 必要な変数が第三者にも伝わる                    |
## 4. 補足知識
### 12 Factor App
「設定は環境変数に持て」という有名な設計原則。Docker・コンテナ界隈での標準的な考え方の根拠になっている。本番環境では同じ考え方で AWS Secrets Manager や GitHub Actions Secrets に差し替えるだけで動く。  
> https://12factor.net/ja/config  
### Docker Compose を使う理由
PostgreSQL・Seq のような外部サービスへの依存があるとき、ローカルへのインストールなしに `docker-compose up` で環境を揃えられる。チームや環境が変わっても構成が再現できる点がメリット。  
### Visual Studio の Docker Compose 統合
ソリューションに `docker-compose.dcproj` が含まれていると、VS が自動的に「Docker Compose」起動プロファイルを追加する。F5 実行すると：  
1. `docker-compose up --build` でコンテナ起動  
2. コンテナ内プロセスにデバッガーを自動アタッチ  
3. ローカルのソースとコンテナ内バイナリを紐付け  
ブレークポイントがコンテナ内のコードに対しても効くのはこの仕組みのため。  

