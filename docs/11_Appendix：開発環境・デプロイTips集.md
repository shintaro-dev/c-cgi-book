## A.0 開発環境の準備（MariaDB / unixODBC）

本書の各章で使用する C 言語 CGI プログラムを動作させるには、以下のソフトウェアが必要です。

- Apache（mod_cgi 有効）
- MariaDB（または MySQL）
- unixODBC（ODBC経由でDB接続するため）
- GCC、make、標準Cライブラリ

### A.0.1 必要パッケージのインストール（Debian/Ubuntu系）

```sh
sudo apt update
sudo apt install apache2 gcc g++ make mariadb-server unixodbc unixodbc-dev libmariadb-dev
```

### A.0.2 MariaDBの初期構築

```sh
sudo mysql_secure_installation
```

- rootパスワードの設定
- 匿名ユーザーの削除
- testデータベースの削除
- リモートrootログインの無効化

#### A.0.2.1 ユーザーとデータベースの作成

```sql
-- MariaDBにログイン
sudo mariadb -u root -p

-- ユーザーとデータベース作成
CREATE DATABASE sample;
CREATE USER 'user'@'localhost' IDENTIFIED BY 'pass';
GRANT ALL PRIVILEGES ON sample.* TO 'user'@'localhost';
FLUSH PRIVILEGES;
```

### A.0.3 unixODBCの導入と確認

#### A.0.3.1 導入確認

```sh
odbcinst -j
```

- 設定ファイルのパスが `/etc/odbc.ini`, `/etc/odbcinst.ini` であることを確認

#### A.0.3.2 接続確認

```sh
isql -v mydb user pass
```

- `odbc.ini` に設定したDSNが正しく認識されていれば接続成功

> ※ `mydb` は DSN 名、`user`, `pass` は第7章で設定したユーザー情報を使ってください。

# 11_Appendix：開発環境・デプロイTips集（）

## A.0 構成とポリシー：この教材が目指す環境とは

このAppendixでは、本書のサンプルCGIプログラムが動作する環境を、Ubuntuベースで一から構築する方法を解説します。開発用のローカル環境としてだけでなく、将来的なVPS・本番運用への橋渡しを見据えた設計になっており、再現性・透明性・学習効果の高さを重視しています。

---

### ■ 使用OSとその理由

本書では、Ubuntu 24.04 LTS（Long Term Support）を前提としています。LTS版は5年間のセキュリティアップデートが提供され、以下のような理由で選定されています：

- 安定性と長期保守の両立が可能
- パッケージが豊富で、開発ツールの導入が容易
- 主要なVPSサービス（さくらのVPS、ConoHa、AWS Lightsail など）でも標準的に提供されている

これにより、ローカルで構築した環境をそのまま本番へ持ち込むことができ、実用的な運用スキルも自然と身につきます。

---

### ■ 本書の構成方針

教材としての明快さと運用面での再現性を両立するため、以下の方針を採用しています：

- **CGIプログラムはC言語で実装**し、動作原理が追いやすい構造とする
- **ODBC接続はDSN定義なし**（接続文字列の直接埋め込み）を基本とし、構成ファイルの依存を最小限に抑える
- **Apacheの基本機能のみを利用**し、特殊なフレームワークやミドルウェアに依存しない
- **MariaDB + unixODBC + Apache + GCC** による軽量構成とし、Ubuntuの標準リポジトリで導入可能な範囲に収める

これにより、構成の理解と再現性が確保され、トラブル時にも自力で対処できる知識が自然と身につきます。

---

### ■ 必要なパッケージ群（Ubuntu）

以下のパッケージを最初に導入しておくと、本書の内容すべてに対応できます：

```bash
sudo apt update
sudo apt install apache2 gcc g++ make mariadb-server unixodbc unixodbc-dev libmariadb-dev
```

---

次節（A.1）からは、Apacheの基本設定やCGI実行設定、MariaDBとの接続、API用のMIME調整など、各機能別に順を追って構築していきます。

---

## A.1 Apacheの基本設定

ApacheにてCGIを有効にするため、以下のような設定を行います。

```bash
sudo a2enmod cgi
sudo systemctl restart apache2
```

デフォルトでは `/usr/lib/cgi-bin/` に `.cgi` ファイルを置くことで実行可能になります。

---

## A.2 UserDir構成の有効化（任意）

ユーザディレクトリでの公開には `mod_userdir` を有効化します。

```bash
sudo a2enmod userdir
sudo systemctl restart apache2
```

設定ファイル：`/etc/apache2/mods-enabled/userdir.conf`

```apache
<Directory /home/*/public_html/cgi-bin>
    Options +ExecCGI
    AddHandler cgi-script .cgi
    Require all granted
</Directory>
```

アクセス例：`http://localhost/~username/cgi-bin/hello.cgi`

---

## A.3 JSON APIに関する設定（第10章対応）

### A.3.1 CGI拡張子の拡張

`.cgi` 以外にも `.api` や `.json.cgi` を使用したい場合：

```apache
AddHandler cgi-script .cgi .api
```

または：

```apache
<Directory "/var/www/html/api">
    Options +ExecCGI
    AddHandler cgi-script .api
</Directory>
```

### A.3.2 MIMEタイプの追加（静的JSONファイル対応）

```apache
AddType application/json .json
```

CGIが出力する `Content-Type` は明示的にプログラム内で指定する必要があります：

```c
printf("Content-Type: application/json; charset=UTF-8\n\n");
```

### A.3.3 クライアント確認用：curl例

```bash
curl -i "http://localhost/cgi-bin/list_diary.api?after_id=10"
```

必要に応じてヘッダを指定：

```bash
curl -H "Accept: application/json" ...
```

---

## A.4 ODBCの構成方針とDSN

本書では DSNなし構成（接続文字列埋め込み）を推奨します：

```c
SQLCHAR connStr[] = "DRIVER={MariaDB};SERVER=localhost;DATABASE=testdb;UID=user;PWD=pass;";
```

### DSNあり接続を使用する場合

`/etc/odbc.ini` または `~/.odbc.ini` にて定義：

```ini
[mydsn]
Driver = /usr/lib/x86_64-linux-gnu/odbc/libmaodbc.so
Server = localhost
Database = testdb
User = user
Password = pass
```

接続コード：

```c
SQLConnect(hdbc, (SQLCHAR*)"mydsn", SQL_NTS, ...);
```

---

## A.5 CGI開発時の罠とTips

### 改行コード問題（Windows→Linux）

```bash
dos2unix hello.cgi
```

### 実行権限の付与

```bash
chmod +x hello.cgi
```

### shebang の確認

```bash
#!/usr/bin/env bash
```

csh や zsh になっていないか注意。

---

## A.6 再現性ある構成のために

- サンプルコードやApache設定をGit管理する
- DSN定義やisqlスクリプトを初期化スクリプトにまとめる
- `/opt/c-cgi-book/` など専用ディレクトリを作成し教材構成を保存

---

## 小まとめ

- Apache側の `AddHandler` や `.htaccess` によって柔軟なAPI設計が可能
- CGIは `Content-Type` を明示出力すること（Apacheは中継するだけ）
- JSON APIのcurlテストやデバッグ手法も把握しておくと開発効率が上がる
- DSNなし／ありの使い分け、UserDir構成など発展的構成も紹介

以上をもとに、本書の各章を補完し、ローカルおよび本番へのデプロイを支援するAppendixとします。

## A.1 mod_cgi の有効化（第2章対応）

Apache で C 言語の CGI プログラムを実行するには、`mod_cgi`（または `mod_cgid`）が有効になっている必要があります。

### A.1.1 有効化手順（Debian / Ubuntu 系）

```sh
sudo a2enmod cgi
sudo systemctl restart apache2
```

- モジュールの有効化後は、Apache の再起動が必要です。

### A.1.2 設定ファイル例

Apache のサイト設定ファイル（`/etc/apache2/sites-available/000-default.conf` など）に以下を追加：

```apacheconf
<Directory "/usr/lib/cgi-bin">
    Options +ExecCGI
    AddHandler cgi-script .cgi .pl .out
</Directory>
```

> ※ 上記の `Directory` パスは、実際に CGI を配置するディレクトリに応じて調整してください。


## A.2 CGIファイルの配置と実行権限（第3章対応）

CGIファイルは Web サーバが実行できる場所に配置し、実行権限を付与する必要があります。

### A.2.1 実行ディレクトリの例

```sh
/usr/lib/cgi-bin/
```

### A.2.2 実行権限の設定

```sh
chmod +x hello.cgi
```

- 所有者およびApacheプロセスが実行できるように設定します。

> ※ SELinuxやAppArmorの設定によっては別途許可が必要な場合があります。


## A.3 Content-Length トラブルとデバッグ（第4章対応）

POSTメソッドでデータを受信する際、`Content-Length` が不正または未設定の場合に `fread()` が正常に動作しないことがあります。

### A.3.1 デバッグ手法

```sh
hexdump -C postdata.txt
```

- 内容を16進で確認し、改行コードや余計な空白が入っていないか確認します。

### A.3.2 テスト送信例（curl）

```sh
curl -X POST -d "name=Alice" http://localhost/cgi-bin/form.cgi
```

- POSTデータの送信確認や、Content-Length の自動付加を確認できます。


## A.4 malloc / free の注意点とデバッグ（第5章対応）

構造体や文字列を動的に確保する場合、`malloc()` や `strdup()` を使用しますが、解放忘れによるメモリリークに注意が必要です。

### A.4.1 確保と解放の基本

```c
char *name = strdup("Alice");
free(name);
```

### A.4.2 メモリリークの検出ツール

```sh
valgrind ./form_handler.cgi
```

- 解放漏れや未初期化のメモリ利用をチェックできます。


## A.5 テンプレートファイルと文字コード（第6章対応）

テンプレートファイルはUTF-8で保存し、改行コードの統一にも注意が必要です。

### A.5.1 配置ディレクトリの例

```sh
/var/www/html/templates/
```

### A.5.2 文字コードの確認

```sh
file template.html
nkf -g template.html
```

- UTF-8 であること、および BOM の有無を確認します。


## A.6 unixODBC 設定と動作確認（第7章対応）

ODBC 接続には `/etc/odbc.ini` と `/etc/odbcinst.ini` の2つの構成ファイルが必要です。

### A.6.1 DSN 設定例（/etc/odbc.ini）

```ini
[mydb]
Driver = MariaDB
Server = localhost
Database = sample
User = user
Password = pass
```

### A.6.2 ドライバ定義例（/etc/odbcinst.ini）

```ini
[MariaDB]
Description = ODBC for MariaDB
Driver = /usr/lib/x86_64-linux-gnu/odbc/libmaodbc.so
```

### A.6.3 接続テスト

```sh
isql mydb user pass
```


## A.7 セッションディレクトリの運用（第8章対応）

セッションファイルを `/tmp/sessions/` などに保存する場合、以下の点に注意します。

### A.7.1 ディレクトリの作成と権限

```sh
sudo mkdir /tmp/sessions
sudo chmod 733 /tmp/sessions
```

- Apache ユーザーが読み書き可能で、他のユーザーからはアクセスできないようにします。

### A.7.2 自動削除対策

一部の Linux では `/tmp` 以下のファイルが自動削除される場合があるため、定期的な確認が推奨されます。


## A.8 簡易CMS用データベース構成（第9章対応）

本書第9章「簡易CMS（日記投稿アプリ）」では、以下のテーブル構成を前提としています。

### A.8.1 diaryテーブルの定義

```sql
CREATE TABLE diary (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    title VARCHAR(255),
    body TEXT,
    is_public TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### A.8.2 初期データの投入例

```sql
INSERT INTO diary (user_id, title, body, is_public)
VALUES
(1, '初めての日記', 'これはテスト投稿です。', 1),
(1, '下書きメモ', 'これは非公開の投稿です。', 0);
```


## A.9 APIテストとContent-Type（第10章対応）

JSON API をテストする際のツールや注意点をまとめます。

### A.9.1 curl による GET / POST 送信

```sh
curl http://localhost/cgi-bin/api.cgi
curl -X POST -H "Content-Type: application/json" -d '{"key":"value"}' http://localhost/cgi-bin/api.cgi
```

### A.9.2 Content-Type の確認

レスポンスに `Content-Type: application/json` が含まれているか確認します。

### A.9.3 整形ツール

```sh
curl http://localhost/cgi-bin/api.cgi | jq
```

- JSONが整形され、ブラウザよりも読みやすくなります。

## A.10 日記投稿アプリのテンプレート対応例（第9章対応）

本章では、第9章「簡易CMS（日記投稿アプリ）」で作成した投稿画面をテンプレート対応に改良した例を紹介します。

> ※ 第8章をご覧の方へ：  
> 本例は、セッション管理によってログイン済みユーザー名を表示する構成にも対応しており、第8章のテンプレート対応にも応用できます。

### A.10.1 テンプレートファイル（template_diary_post.html）

```html
<html>
<head><title>${title}</title></head>
<body>
<h1>${title}</h1>
<p>ようこそ、${username}さん。</p>
<form method="POST" action="/cgi-bin/diary_post.cgi">
  <textarea name="body" rows="5" cols="40"></textarea><br>
  <input type="submit" value="投稿">
</form>
</body>
</html>
```

### A.10.2 diary_post.cgi（テンプレート対応版）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "template.h"
#include "session.h"

int main(void) {
    session_t session;
    session_start(&session);

    char *template = load_template("template_diary_post.html");
    replace(template, "${title}", "日記を書く");
    replace(template, "${username}", session.username);

    printf("Content-Type: text/html

");
    puts(template);

    free(template);
    session_free(&session);
    return 0;
}
```

### A.10.3 template.h の関数例（抜粋）

```c
char *load_template(const char *filename);
void replace(char *template, const char *key, const char *value);
```

### A.10.4 効果と利点

- HTML構造とロジックの分離により、保守性と再利用性が大幅に向上します。
- テンプレートファイルを差し替えるだけで画面デザインを変更できます。

## A.11 セッションセキュリティ補足（第9章対応）

セッションの安全性を高めるための具体的な実装例を以下に示します。

### A.11.1 セッションIDの生成（/dev/urandom利用）

```c
#include <stdio.h>
#include <stdlib.h>

void sid_generate(char *sid, size_t len) {
    FILE *fp = fopen("/dev/urandom", "rb");
    fread(sid, 1, len, fp);
    fclose(fp);
    for (size_t i = 0; i < len; i++) {
        sid[i] = "abcdefghijklmnopqrstuvwxyz0123456789"[sid[i] % 36];
    }
    sid[len] = '\00';
}
```

### A.11.2 Cookieの安全な出力

```c
printf("Set-Cookie: SID=%s; HttpOnly; Secure; SameSite=Strict
", sid);
```

> ※ `Secure` 属性は HTTPS 通信でのみ有効です。本番環境ではHTTPSを利用してください。