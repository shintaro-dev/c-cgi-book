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
