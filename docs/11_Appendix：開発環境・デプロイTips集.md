# 11_Appendix：開発環境・デプロイTips集 (Draft)

## A.1 環境全体像と構成方針

...（既存内容省略）...

## A.2 OSの準備

この節では、教材で利用するC言語CGIプログラムを動作させるために必要な基本的なOS環境（Ubuntu 24.04 LTS）と、開発・実行に必要なパッケージ群をインストールする手順を示します。

本書では、Ubuntu 24.04 LTS（Long Term Support）を前提とします。LTS版はセキュリティ更新が5年間提供されるため、長期運用にも耐えうる安定性があります。また、主要なVPSサービス（さくらのVPS、ConoHa、AWS Lightsailなど）でも採用されており、ローカル開発から本番環境への移行もスムーズです。

### A.2.1 パッケージ一覧
以下のパッケージをインストールします：

- `build-essential`：gcc, make 等、Cの基本ビルド環境
- `gcc`, `gdb`：C言語コンパイラとデバッガ
- `apache2`：Webサーバ（CGI実行用）
- `mariadb-server`：データベース本体
- `unixodbc`：ODBCドライバマネージャ
- `libodbc1`, `libodbc-dev`：ODBC開発用ヘッダ類
- `curl`, `vim`, `man-db` など：補助ツール（任意）

### A.2.2 インストール手順
以下のコマンドを端末で順に実行してください。

```bash
# パッケージリストを最新化
sudo apt update

# 必須パッケージのインストール
sudo apt install -y \
  build-essential gcc gdb \
  apache2 \
  mariadb-server \
  unixodbc libodbc1 libodbc-dev \
  curl vim man-db
```

※一部パッケージは既にインストール済みの場合があります（特に`curl`, `vim`）。その場合も問題ありません。

### A.2.3 動作確認
以下のコマンドで主要なコンポーネントが正しくインストールされたか確認します。

```bash
# gccのバージョン確認
gcc --version

# Apacheのステータス確認
sudo systemctl status apache2

# MariaDBのステータス確認
sudo systemctl status mariadb

# ODBCの動作確認（空リストが出ればOK）
odbcinst -q -d
```

すべてが正常に動作していれば、以降の章で扱うCGIやデータベース連携の準備が整っています。

---

## A.3 Apache HTTP Serverの設定

...（以降続く）...