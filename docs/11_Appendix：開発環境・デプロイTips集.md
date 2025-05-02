# 11_Appendix：開発環境・デプロイTips集 (Draft)

## A.1 環境全体像と構成方針

このAppendixでは、本書の学習を支える開発環境・実行環境の構築手順を整理し、読者が自身の手元でサンプルコードを動作させられることを目指します。

教材で扱う構成はできる限りシンプルに留めつつも、現実的なWebアプリケーション開発の流れに沿った形になっており、最終的にはVPS等の本番環境にデプロイ可能な構成も視野に入れています。

---

### 想定環境（基本構成）

- OS：Ubuntu 24.04 LTS（Long Term Support）
- Webサーバ：Apache HTTP Server（CGI対応）
- データベース：MariaDB
- CGI実行：C言語でビルドした実行ファイル
- DB接続方式：ODBC（MariaDB Connector/ODBC＋unixODBC）

---

### 構成ポリシー

- **学習コストを抑えつつも、実運用に繋がる構成**  
  複雑なフレームワークは導入せず、Webサーバ・データベース・CGIプログラムがそれぞれどう連携して動いているかを、できるだけ「素の状態」で体験できる構成を採用しています。

- **Ubuntu LTSを採用する理由**  
  Ubuntu 24.04は長期サポート（LTS）対象であり、セキュリティ更新が5年間提供されます。VPSサービスでも広く採用されており、開発からデプロイまでの接続がスムーズです。  
  他にもCentOS系やAlpine Linux等のディストリビューションは存在しますが、汎用性・採用実績・学習コストの観点からUbuntuを採用しています。

- **DSNなしのODBC接続を基本とする理由**  
  DSN（Data Source Name）を登録する方式は便利ですが、システム依存・ユーザー依存の設定になりやすく、再現性や移植性に課題が残ります。本書では接続文字列をソースコードに記述する「DSNなし接続方式（Driver=〜; Server=〜;）」を基本とします。

- **VMやVPSの両方で再現可能な構成**  
  本書ではローカル開発環境として VirtualBox 等でUbuntuを動かす構成を基本としつつ、最終的には VPS（さくらのVPS、ConoHa、Lightsail等）にそのまま移行可能な構成となっています。

---

この方針に基づいて、以下の章では OS／Apache／MariaDB／ODBC／CGIビルド環境などの準備手順を段階的に解説します。

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

この節では、C言語CGIプログラムを動作させるために、Apache HTTP Server の基本設定を行います。

### A.3.1 ApacheのCGI有効化

デフォルトのApacheはCGIを無効化しているため、`cgi` モジュールを明示的に有効化する必要があります。

```bash
sudo a2enmod cgi
sudo systemctl restart apache2
```

---


### A.3.2 CGIファイルの配置と実行ディレクトリ

UbuntuのApacheでは、標準で `/usr/lib/cgi-bin/` がCGI実行ディレクトリとして設定されています。  
このディレクトリは、Apacheの初期設定で `ExecCGI` が許可されており、特別な設定変更をしなくても `.cgi` ファイルが実行可能です。

以下のようにして、CGIプログラムをビルド・配置・実行権限付与します。

```bash
# Cソースをビルド
gcc hello.c -o hello.cgi

# 標準のCGIディレクトリに配置
sudo mv hello.cgi /usr/lib/cgi-bin/
sudo chmod 755 /usr/lib/cgi-bin/hello.cgi
```

ブラウザで以下にアクセスすると、CGIの実行結果が確認できます：

```
http://localhost/cgi-bin/hello.cgi
```

---


### A.3.3 Apache設定ファイルの確認

UbuntuにおけるApacheのデフォルト設定では、`/usr/lib/cgi-bin/` に対してすでにCGIの実行が許可されています。

確認のため、以下のディレクトリ設定が `/etc/apache2/sites-available/000-default.conf` または `/etc/apache2/conf-enabled/serve-cgi-bin.conf` に含まれていることをチェックしてください。

```apacheconf
<Directory "/usr/lib/cgi-bin">
    Options +ExecCGI
    SetHandler cgi-script
    Require all granted
</Directory>
```

この設定がすでに有効であれば、追加の設定変更は不要です。

設定変更を行った場合は、Apacheを再起動して反映してください。

```bash
sudo systemctl restart apache2
```


---


### A.3.4 動作確認用サンプルCGI

`hello.cgi` を以下のように作成します。

```c
#include <stdio.h>

int main(void) {
    printf("Content-Type: text/html\n\n");
    printf("<html><body><h1>Hello, CGI!</h1></body></html>\n");
    return 0;
}
```

ビルド＆配置：

```bash
gcc hello.c -o hello.cgi
sudo mv hello.cgi /usr/lib/cgi-bin/
sudo chmod 755 /usr/lib/cgi-bin/hello.cgi
```

ブラウザでアクセス：

```
http://localhost/cgi-bin/hello.cgi
```


---

### A.3.5 よくあるエラー

| 症状 | 対処 |
|------|------|
| 403 Forbidden | パーミッション不足、実行権限なし |
| 500 Internal Server Error | CGI内エラー、`Content-Type` 出力忘れ |
| 404 Not Found | パス設定ミス、ファイル配置場所の間違い |

ログ確認は以下で：

```bash
sudo tail -f /var/log/apache2/error.log
```

---

### 小まとめ

- `a2enmod cgi` でCGIモジュールを有効化
- CGI実行用ディレクトリを作成・許可
- Apache設定ファイルで `ExecCGI` を許可
- 確認用 `.cgi` をビルド＆ブラウザで動作確認


## A.4 MariaDBのセットアップ

この節では、CGIプログラムからのデータベース接続に必要な MariaDB の基本的なセットアップ手順を解説します。対象環境は Ubuntu 24.04 LTS を前提とし、ローカル環境および本番デプロイの両方で再現可能な最小構成を目指します。

### A.4.1 パッケージのインストール

MariaDBはDebian/Ubuntu系で公式にサポートされており、以下のコマンドでインストールできます。

```bash
sudo apt update
sudo apt install -y mariadb-server mariadb-client
```

インストール後は MariaDB サービスを起動・有効化しておきます。

```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb
```

### A.4.2 セキュリティ設定（mysql_secure_installation）

初期インストール後は、`mysql_secure_installation` による初期設定を行います。

```bash
sudo mysql_secure_installation
```

#### 主な確認・設定ポイント：

- root パスワードの設定（または現在のUnix socket認証を維持）
- 匿名ユーザの削除
- テストデータベースの削除
- リモートrootログインの禁止

> ※ 読者が詰まりやすいポイント：
> ODBC経由で接続する場合、Unix socket 認証ではログインできないため、
> 明示的に `mysql_native_password` を使うよう root を再設定する必要があります。

例：

```sql
-- mysql (または mariadb) シェルにて
ALTER USER 'root'@'localhost' IDENTIFIED VIA mysql_native_password USING PASSWORD('yourpassword');
FLUSH PRIVILEGES;
```

※ `sudo mariadb` でrootユーザとしてログインできます。

---

### A.4.3 アプリケーション用データベースとユーザの作成

本書で使用するテストアプリ用に、次のような初期SQLスクリプトを実行します：

```sql
-- アプリケーション用データベースとユーザ作成
CREATE DATABASE testapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'webuser'@'localhost' IDENTIFIED BY 'webpass';

GRANT ALL PRIVILEGES ON testapp.* TO 'webuser'@'localhost';
FLUSH PRIVILEGES;
```

SQL実行方法（MariaDBシェル内で手打ち or SQLファイルからバッチ実行）：

```bash
sudo mariadb < setup.sql
```

---

### A.4.4 動作確認（CLI）

設定が正しくできているか、次のようにログインして確認します。

```bash
mariadb -u webuser -p testapp
```

ログイン成功後、簡単なテーブルを作って `SELECT` できればOKです。

---

### 小まとめ

- `mariadb-server`, `mariadb-client` をインストール
- `mysql_secure_installation` 実行後、必要に応じて `mysql_native_password` に切り替え
- アプリ用DBとユーザーを作成（`webuser` / `testapp`）
- CLIでの接続確認まで行う

次章 A.5 では、この設定をもとに ODBC からの接続確認を行います。

## A.5 ODBCドライバの設定

この節では、CGIプログラムからMariaDBに接続するためのODBCドライバ設定について解説します。本書では **DSNなし接続方式** を基本とし、開発・運用の再現性と移植性を重視した構成を採用しています。

---

### A.5.1 unixODBCとMariaDB Connector/ODBCのインストール

まずはODBCの基本モジュール（unixODBC）とMariaDB用のドライバ（Connector/ODBC）をインストールします。

```bash
sudo apt update
sudo apt install -y unixodbc unixodbc-dev odbcinst libodbc1 libmariadb-dev libmariadb3
```

MariaDB Connector/ODBCの実体ファイルは通常以下の場所にインストールされます：

```
/usr/lib/x86_64-linux-gnu/odbc/libmaodbc.so
```

---

### A.5.2 ドライバ定義ファイル（odbcinst.ini）の登録

`odbcinst.ini` にドライバエントリを登録します。次のようにファイルに記述するか、コマンドで一括登録できます。

#### 手動記述（`/etc/odbcinst.ini`）の例：

```ini
[MariaDB]
Description = MariaDB ODBC driver
Driver = /usr/lib/x86_64-linux-gnu/odbc/libmaodbc.so
```

#### コマンドによる登録：

```bash
sudo odbcinst -i -d -f /etc/odbcinst.ini
```

このコマンドにより `odbcinst -q -d` でドライバ名が列挙されるようになります。

---

### A.5.3 DSNなし接続の推奨（本書の基本方針）

ODBCには「DSN（Data Source Name）」という名前付き接続設定がありますが、本書では以下の理由により **DSNなし接続** を推奨しています。

- 設定ファイルが不要で、**プログラムに設定が自己完結する**
- ユーザ依存・OS依存の差異が減り、**再現性・可搬性が高い**
- VPS・クラウド環境での自動デプロイがしやすい

#### DSNなし接続の例（接続文字列）：

```c
"Driver=MariaDB;Server=localhost;Database=testapp;User=webuser;Password=webpass;"
```

この文字列を `SQLDriverConnect` に直接渡す形で使用します。

---

### A.5.4 DSNあり接続の参考（odbc.ini）

参考として、DSNを使う場合の設定ファイル例を示します。  
`/etc/odbc.ini` または `~/.odbc.ini` に以下のように記述します：

```ini
[testdsn]
Driver = MariaDB
Server = localhost
Database = testapp
User = webuser
Password = webpass
```

登録確認：

```bash
odbcinst -q -s
```

---

### A.5.5 接続確認：`isql` コマンドによるテスト

ODBCの接続確認には `isql` コマンドが便利です。以下のように実行します。

#### DSNなしで接続確認：

```bash
isql -v "Driver=MariaDB;Server=localhost;Database=testapp;User=webuser;Password=webpass;"
```

#### DSNありで接続確認：

```bash
isql -v testdsn webuser webpass
```

---

### よくあるエラーと対処

| エラー例 | 原因と対処 |
|----------|------------|
| `[08001] Could not connect` | Server名・ポートの指定ミス／ユーザ認証ミス |
| `isql: symbol lookup error` | 32bit/64bit混在のライブラリ競合。`libmaodbc.so`の場所を確認 |
| `Segmentation fault` | `Driver=` 記述ミス or 不正なINIファイル構文 |

---

### 小まとめ

- unixODBCとMariaDB Connector/ODBCをインストール
- `odbcinst.ini` にMariaDBドライバを定義
- DSNなし接続を推奨（構成ファイル不要で再現性が高い）
- `isql` で動作確認し、接続に成功すれば準備完了

この設定が完了すれば、次章でCGIからODBC経由でDBアクセスが可能になります。