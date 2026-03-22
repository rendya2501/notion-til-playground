---
title: "Docker 勉強メモ その1"
type: "Tech"
description: "Dockerの基本概念をC#/.NETのビルドプロセスに対応させて整理。Dockerfile・イメージ・コンテナの関係（.cs→.dll→プロセス）、docker-composeの仕組み、マルチステージビルド、Volumeによるデータ永続化、コンテナ間ネットワークの基礎を網羅。

  • https://claude.ai/chat/d5a28093-5448-468c-af86-9eddf5e83094"
tags: ["Docker"]
date: "2026-03-22T00:00:00"
---

## 核心的なアナロジー
Dockerの概念は静的言語のビルドプロセスとそのまま対応する。  
| Docker         | C# / .NET             |
| -------------- | --------------------- |
| Dockerfile     | クラス定義（.cs ソースコード）     |
| `docker build`   | コンパイル（`dotnet build`）   |
| イメージ           | .dll（コンパイル済みバイナリ）     |
| `docker run`   | 実行（`dotnet run`）      |
| コンテナ           | 動いているプロセス             |
Dockerfileはテキストファイルで直接実行できない。`docker build` でコンパイルして初めて動かせる形（イメージ）になる。この関係が `.cs` → `.dll` と同じ。  
---
## 登場人物
### Dockerfile
イメージを作るための手順書（ソースコード）。各命令が「レイヤー」として積み重なる。レイヤーはキャッシュされるので、変更頻度の低いものを上に書くのが鉄則。  
### 主要な命令
| 命令           | 意味                                                            |
| ------------ | ------------------------------------------------------------- |
| `FROM`       | ベースイメージを指定（土台となるOS・ランタイム）                                     |
| `WORKDIR`    | 作業ディレクトリを指定。以降の命令はここを起点にする                                    |
| `COPY`       | ホストのファイルをコンテナにコピー                                             |
| `RUN`        | ビルド時にコマンドを実行（パッケージインストール・ビルドなど）                               |
| `EXPOSE`     | コンテナが使うポートを宣言（実際の公開は `docker run -p` や compose の `ports` で行う）   |
| `ENTRYPOINT`   | コンテナ起動時に実行するコマンド                                              |
| `ARG`        | ビルド時に外から渡せる変数                                                 |
| `ENV`        | コンテナ内の環境変数を設定                                                 |
| `AS`         | ステージに名前をつける（マルチステージビルドで使う）                                    |
### 記述例
```docker
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG BUILD_CONFIGURATION=Release   # デフォルト値あり。外から上書き可能
WORKDIR /src

# csprojだけ先にCOPYしてrestoreする → 依存パッケージが変わらなければキャッシュが効く
COPY ["src/Web.Api/Web.Api.csproj", "src/Web.Api/"]
COPY ["src/Infrastructure/Infrastructure.csproj", "src/Infrastructure/"]
RUN dotnet restore "./src/Web.Api/Web.Api.csproj"

COPY . .   # ソースコード全体をコピー（restoreの後にする理由はキャッシュ最適化のため）
WORKDIR "/src/src/Web.Api"
RUN dotnet build "./Web.Api.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
RUN dotnet publish "./Web.Api.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .   # publishステージの成果物だけコピー
ENTRYPOINT ["dotnet", "Web.Api.dll"]
```
---
### イメージ
- `docker build` の成果物（バイナリ）  
- 実体は「ファイルシステムのスナップショット」。OSのファイル群・ランタイム・アプリのコードなどが全部まとまったzipのようなもの  
- read-only（読み取り専用）、SSD/HDDのディスク上に保存される  
- DockerHubから取得（`docker pull`）するか、Dockerfileから自前でビルドする  
- 明示的に削除しない限りディスクに残り続け、次回ビルド時に再利用される  
- Docker Desktop の「Images」タブで確認できる  
---
### コンテナ
- イメージから作られた動いている実体（プロセス）  
- `docker run` でイメージがメモリに展開されてプロセスとして動く（= new + 実行）  
- イメージの上に書き込み可能なレイヤーが追加される。アプリが書いたデータはここに乗る  
- 同じイメージから何個でも作れて、それぞれ独立している  
- 停止・再開ではデータは消えない。削除したときだけデータが消える  
- Docker Desktop の「Containers」タブで確認・起動・停止できる  
```plain text
[コンテナA の書き込みレイヤー]  ← Aだけの状態
[コンテナB の書き込みレイヤー]  ← Bだけの状態
━━━━━━━━━━━━━━━━━━━━━━━━
[イメージ（read-only）]         ← 共有・不変
```
イメージとコンテナの流れ：  
```plain text
ディスク：イメージを保存（倉庫）
　↓ docker run
メモリ：コンテナとして展開されて動く（作業場）
```
---
## docker-compose
複数のコンテナをまとめて管理するツール。Docker Desktop をインストールすると `docker compose` コマンドが一緒に入る。  
`docker-compose.yml` はそのツールが読み込む設定ファイル。ツールがなければymlを書いても何も起きない。関係性は `git` コマンドと `.gitconfig` と同じで、ツールあってこその設定ファイル。  
`docker compose up` を実行すると内部で以下が自動で走る。  
```plain text
docker compose up
  └─ docker build   （Dockerfile → イメージ）  ← 初回のみ
  └─ docker run     （イメージ → コンテナ起動）
```
2回目以降はイメージのビルドをスキップして既存コンテナをそのまま起動するので速い。  
### docker-compose.yml の書き方
```yaml
services:
  サービス名:                          # コンテナの識別子。他のコンテナからホスト名としても使える
    image: イメージ名:タグ             # DockerHubのイメージを使う場合
    build:                             # 自前ビルドの場合（imageと排他）
      context: .                       # Dockerfileを探すルートディレクトリ
      dockerfile: src/Web.Api/Dockerfile
    container_name: 任意の名前         # Docker Desktop上での表示名
    environment:                       # 環境変数を渡す
      - KEY=VALUE
      - ASPNETCORE_ENVIRONMENT=Development
    ports:
      - "ホスト側ポート:コンテナ側ポート"   # 外からアクセスできるようにする
    volumes:
      - ホスト側パス:コンテナ側パス    # データの永続化やファイルの共有
      - ホスト側パス:コンテナ側パス:ro # roをつけると読み取り専用
    depends_on:
      - 他のサービス名                 # 起動順序の制御（起動を待つわけではない）
```
### ポートマッピングの読み方
```yaml
ports:
  - "5000:8080"
#    ↑     ↑
# ホスト  コンテナ
# 外から  中では
# 見える  この番号
```
ブラウザで `localhost:5000` にアクセスすると、コンテナ内の8080番に転送される。コンテナ内のアプリは8080で待ち受けているので繋がる。  
### 環境変数の役割
コンテナ内のアプリに設定を渡す仕組み。DB接続文字列や動作モード（Development/Production）などを外から注入できる。ハードコードを避けてコンテナを使い回せるようにするための設計。  
```yaml
environment:
  - ASPNETCORE_ENVIRONMENT=Development  # ASP.NETの動作環境を指定
  - ASPNETCORE_HTTP_PORTS=8080          # 待ち受けポートを指定
  - POSTGRES_DB=mydb                    # PostgreSQL固有の初期化設定
  - POSTGRES_USER=postgres
  - POSTGRES_PASSWORD=postgres
```
### docker-compose.override.yml
`docker compose up` 実行時に `docker-compose.yml` と自動マージされる。開発専用の設定を分離するために使う。本番では `docker-compose.yml` だけを明示的に指定して使う。  
### よく使うコマンド
```bash
# サービスを名指しで起動（サービス名はymlのservices:に書いた名前）
docker compose up -d postgres seq

# 全サービスを起動
docker compose up -d

# 停止（コンテナは残る）
docker compose stop

# 停止してコンテナを削除
docker compose down

# コンテナ・ボリュームごと全削除（DBデータも消える）
docker compose down -v

# ログを見る
docker compose logs -f
```
---
## ネットワーク
`docker compose up` を実行すると、docker-composeは**内部ネットワークを自動生成**する。同じcompose内のサービスはこのネットワーク内でお互いに通信でき、**サービス名がそのままホスト名**として使える。  
```yaml
# appsettings.Development.json の接続文字列
"Database": "Host=postgres;Port=5432;Database=clean-architecture"
#                  ↑
#          サービス名がホスト名として解決される
```
外部（ブラウザなど）からはポートマッピングで公開されたポートを通じてのみアクセスできる。  
```plain text
外部（ブラウザ）
  → localhost:5000（ホスト側）
    → コンテナ:8080（ポートマッピング）

コンテナ間通信
  → web-api → postgres:5432（内部ネットワーク、サービス名で解決）
```
---
## マルチステージビルド
本番イメージに不要なものを含めないための手法。  
| イメージ            | 用途               | ディスクサイズ |
| --------------- | ---------------- | ------- |
| `dotnet/sdk`    | ビルドに使う（コンパイラ等含む）   | 約800MB   |
| `dotnet/aspnet`   | 実行だけできればいい       | 約200MB   |
ビルドにはSDKが必要だが、完成したアプリを動かすだけならSDKは不要。「SDKイメージでビルドして、成果物だけ軽いaspnetイメージにコピーする」という二段構えにすることで、本番イメージを軽くできる。  
この仕組みを使わない場合、PC本体にSDKをインストールしてビルドする必要がある。Dockerを使うことでそのインストール作業ごとコンテナ内で完結し、PC本体を汚さずに済む。  
```docker
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build   # SDKイメージでビルド
RUN dotnet publish -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final # 軽いランタイムイメージが土台
COPY --from=build /app/publish .                    # 成果物だけコピー
ENTRYPOINT ["dotnet", "Web.Api.dll"]
```
---
## Volume（永続化）
### Volumeとは
コンテナは使い捨てが前提で、削除するとコンテナ内に書き込まれたデータが全部消える。Volumeはデータをコンテナの外（ホスト側）に逃がす仕組み。  
```yaml
volumes:
  - ./.containers/db:/var/lib/postgresql/data
#   ホスト側のパス  :  コンテナ側のパス
```
PostgreSQLは `/var/lib/postgresql/data` にデータを書き込もうとするが、そこがホストの `.containers/db` にひも付いているので実際の書き込み先はホスト側になる。コンテナを削除してもホスト側のフォルダは残るのでデータが生き残る。  
パスの紐づけはシンボリックリンクに近い概念で、「このパスへのアクセスは別の場所に転送する」というOSレベルのマウントの仕組み。  
### データが消えるタイミング
```plain text
停止 → 再開：データ残る（コンテナの書き込みレイヤーが保持される）
削除 → 再作成：データ消える（Volumeがなければ）
削除 → 再作成：データ残る（Volumeがあれば）
```
### VolumeのファイルシステムとホストOS
コンテナの中はLinux環境なので `/var/lib/...` というLinuxのパス体系で構成されている。ホスト側はWindowsなので `C:\Users\...` のパスになるが、Dockerがその差を吸収してマウントしてくれる。VolumeはLinuxの通常のファイルシステムと同じ階層構造を持つ。  
Docker Desktop の「Volumes」タブで確認・削除できる。  
---
## 用語メモ
### ホスト（host）
英語の host は「何かを収容・受け入れる側」という意味。パーティのホスト（場を提供する人）と同じ語源。  
ITにおいては「他のソフトウェアやサービスを動かす土台となるマシンやOS」を指す。Dockerの文脈ではコンテナを動かしているPC・OSがホスト。  
```plain text
ホストOS（Windows）← 自分のPC
  └─ Docker
       └─ コンテナ（Linux）← ゲスト側
```
### ホスト vs プロバイダー
- **ホスト**：何かを動かしている「場所・土台」に着目した言葉  
- **プロバイダー**：何かを「提供する役割」に着目した言葉  
同じものが文脈によって両方の呼ばれ方をすることもある。AWSはクラウドサービスを提供するのでプロバイダー。そのAWSのサーバー上でDockerを動かしているなら、そのサーバーはDockerにとってのホストでもある。  

