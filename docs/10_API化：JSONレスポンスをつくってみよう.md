# 10. API化：JSONレスポンスをつくってみよう

## 10.1 APIとは何か？JSONとは？

これまでの章では、HTMLを返すCGIを中心に扱ってきました。  
フォームを送信し、HTMLとして結果を受け取る… これがいわゆる「Webページ」です。

しかし、**WebアプリケーションがWebページだけを相手にしている時代は終わりました**。

- スマートフォンアプリ
- JavaScript（Ajax）を使った動的ページ
- 別のサーバからアクセスするクライアント

これらは、**「HTMLではなくデータそのもの（JSON形式など）」を求めています。**  
そんなときに必要なのが **API（Application Programming Interface）** です。

---

### ■ APIとは？

APIとは「アプリケーションから使うためのインタフェース」のこと。  
Webの世界でのAPIは、通常HTTPを使ってデータをやりとりする仕組みを指します。

たとえば：

- 「IDが100より新しい日記一覧をください」  
  `GET /cgi-bin/api/entries.cgi?after_id=100`

- 「この投稿内容を登録してください」  
  `POST /cgi-bin/api/post_entry.cgi`

といったリクエストに対して、CGIプログラムが**HTMLではなくJSON形式でレスポンスを返す**のが典型的なAPIの姿です。

---

### ■ JSONとは？

JSON（JavaScript Object Notation）は、  
**人にも読みやすく、プログラムでも扱いやすい軽量データ形式**です。

以下のような構造をしています：

```json
{
  "id": 101,
  "title": "はじめての日記",
  "body": "今日はC言語でCGIを書きました。",
  "created_at": "2025-05-03 14:00:00"
}
```

複数件のデータを含むときは配列になります：

```json
[
  { ... },
  { ... }
]
```

C言語には標準でJSONを扱う機能はありませんが、  
**文字列操作とエスケープ処理を工夫すれば、十分に自力で出力できます。**

---

### ■ この章で目指すこと

この章では、次のようなことを実現します：

- `Content-Type: application/json` を返すCGIを作る
- データベース（ODBC）から取得した内容をJSONとして出力する
- GETパラメータで「after_id」を指定して新着データを取得する
- JSONの構造とエスケープ処理の基本を学ぶ

---

> CGIでも、APIはつくれる。  
> 軽量でシンプルなCの力で、データを届けよう。

## 10.2 CGIでJSONレスポンスを返すには

HTMLを返すCGIと、JSONを返すCGIの違いは意外とシンプルです。  
**「Content-Type」を適切に設定して、構造化データを出力する**だけで、ブラウザやクライアント側はJSONとして扱ってくれます。

---

### ■ JSONレスポンスの基本構造

CGIでは、最初にHTTPレスポンスのヘッダを出力する必要があります。  
HTMLなら `Content-Type: text/html` を出力していましたが、JSONでは次のようにします：

```c
printf("Content-Type: application/json\n\n");
```

この1行で、「これから出力するのはJSONだよ」とクライアントに伝えることができます。  
あとは、`printf()` などを使ってJSON形式の文字列を出力すればOKです。

---

### ■ 最小構成のJSON出力CGI

以下は、固定された内容をJSONで出力するだけの最小構成のCGIです。

```c
#include <stdio.h>

int main(void) {
    printf("Content-Type: application/json\n\n");

    printf("{\n");
    printf("  \"message\": \"Hello, JSON!\",\n");
    printf("  \"status\": \"success\"\n");
    printf("}\n");

    return 0;
}
```

---

### ■ 実行例（curlで確認）

このCGIを `hello_json.cgi` としてビルド＆設置したら、以下のように確認できます：

```bash
curl http://localhost/cgi-bin/hello_json.cgi
```

出力：

```json
{
  "message": "Hello, JSON!",
  "status": "success"
}
```

---

### ■ JSONは「構造」に意味がある

JSONの魅力は、ただの文字列ではなく**構造化されたデータ**であることです。

- 配列 → `[...]`
- オブジェクト → `{ "key": value }`
- 数値、文字列、論理値、null などが使える

HTMLとは異なり、**画面表示のためではなく「プログラムが読み取るための形式」**だということを意識しておきましょう。

---

### ■ 文字列のエスケープに注意！

C言語でJSONを手動で組み立てる場合、次のような点に注意が必要です：

- ダブルクォート `"` → `\"` にエスケープ
- バックスラッシュ `\` → `\\`
- 改行 `\n` → JSONとしては文字列扱い（不要なら削除）

たとえば次のようなコードはNGです：

```c
printf("{ "name": "Taro" }"); // ← ダブルクォートが混乱する
```

正しくは：

```c
printf("{ \"name\": \"Taro\" }");
```

本章では、**データベースの値をprintfで出力するだけ**という方法を使うため、  
**値の文字列エスケープ関数を自作して使い回す**のが基本になります。

---

### ■ 小まとめ

- `Content-Type: application/json` を出力することでJSONレスポンスが可能
- JSONの構造を理解し、C言語で組み立てるときはエスケープに注意
- 最小CGIでJSON出力を確認し、次に「DBの内容をJSON化」へ進む

次は、ODBC経由で日記データを取得し、それを**配列形式のJSONで出力する実装**に取り組みます！

## 10.3 ODBCで取得したデータをJSONに変換する

ここでは、ODBCを使ってデータベースから日記データを取得し、  
それを**JSON形式の配列データとして出力するCGI**を実装していきます。

---

### ■ 想定するテーブル構造

前章までで使用した `entries` テーブル（または `sample_table`）を前提とします。  
以下のような構造を想定します：

```sql
CREATE TABLE entries (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255),
    body TEXT,
    created_at DATETIME
);
```

---

### ■ 出力されるJSONの形式（例）

```json
[
  {
    "id": 101,
    "title": "はじめての日記",
    "body": "C言語でCGI書いた！",
    "created_at": "2025-05-03 14:00:00"
  },
  {
    "id": 102,
    "title": "今日の発見",
    "body": "ODBC便利すぎる。",
    "created_at": "2025-05-03 16:45:12"
  }
]
```

---

### ■ 処理の流れ

1. 環境変数から `QUERY_STRING` を取得し、`after_id` の値を取得（オプション）
2. ODBCを使ってデータベースに接続
3. SQL文を実行し、`id > ?` の条件付きでSELECT
4. 結果セットを1件ずつ読み取り、JSONオブジェクトとして出力
5. 全体を `[` と `]` で囲んで配列に整形

---

### ■ JSONエスケープ用の補助関数（例）

```c
void json_escape(const char *src, char *dest, size_t size) {
    size_t i = 0;
    while (*src && i < size - 1) {
        if (*src == '\\') {
            if (i < size - 2) dest[i++] = '\\';
            dest[i++] = '\\';
        } else if (*src == '"') {
            if (i < size - 2) dest[i++] = '\\';
            dest[i++] = '"';
        } else if (*src == '\n') {
            if (i < size - 2) dest[i++] = '\\';
            dest[i++] = 'n';
        } else {
            dest[i++] = *src;
        }
        src++;
    }
    dest[i] = '\0';
}
```

これは簡易版です。本格的には `\r`, `\t`, 制御文字なども考慮すべきです。

---

### ■ JSON出力のコツ

- 最初の要素の前には `[` を出力
- 各オブジェクトの間には `,` が必要（最後の要素の後には不要）
- 最後に `]` を出力

これらを処理するために、1件目かどうかを判定するフラグ（`first = 1`）を使うと良いでしょう。

---

### ■ 実装イメージ

```c
printf("Content-Type: application/json\n\n");
printf("[\n");

int first = 1;
while (SQLFetch(hStmt) != SQL_NO_DATA) {
    // データ取得・エスケープ処理…

    if (!first) printf(",\n");
    first = 0;

    printf("  { \"id\": %d, \"title\": \"%s\", \"body\": \"%s\", \"created_at\": \"%s\" }",
           id, escaped_title, escaped_body, created_at);
}

printf("\n]\n");
```

---

### ■ 小まとめ

- SELECT結果をJSON形式に手動で変換するには、**文字列の構造とルールを正確に把握**する必要がある
- C言語では、値のエスケープ処理を**必ず自前で対応**する必要がある
- 配列形式で出力するには、**最初の要素かどうか**を追跡する必要がある

次は、`after_id` パラメータを使った「新着のみ取得」の実装に挑戦していきます！

## 10.4 GETパラメータ `after_id` を使った新着取得

APIとして便利に使ってもらうには、常にすべてのデータを返すのではなく、  
**「指定されたIDより新しいデータだけを返す」**といったフィルタリング機能が必要です。

ここでは、GETパラメータ `after_id` を使って、新着エントリのみを返すCGIを実装します。

---

### ■ クエリパラメータの取得

`after_id` の値は、`QUERY_STRING` に含まれているため、次のように取得できます：

```c
#include <stdlib.h>
#include <string.h>

int get_after_id() {
    char *query = getenv("QUERY_STRING");
    if (!query) return 0;

    char *after_id_str = strstr(query, "after_id=");
    if (!after_id_str) return 0;

    after_id_str += strlen("after_id=");
    return atoi(after_id_str); // 数値化
}
```

---

### ■ SQL文に組み込む

取得した `after_id` をSQLに組み込みます（バインド変数を使う場合）：

```c
SQLINTEGER after_id = get_after_id();

char *sql = "SELECT id, title, body, created_at FROM entries WHERE id > ? ORDER BY id ASC";
SQLPrepare(hStmt, (SQLCHAR *)sql, SQL_NTS);
SQLBindParameter(hStmt, 1, SQL_PARAM_INPUT, SQL_C_LONG, SQL_INTEGER, 0, 0, &after_id, 0, NULL);
```

または文字列に直接埋め込む方法もあります（バインドが難しい場合）：

```c
char sqlbuf[256];
snprintf(sqlbuf, sizeof(sqlbuf),
         "SELECT id, title, body, created_at FROM entries WHERE id > %d ORDER BY id ASC",
         after_id);
SQLExecDirect(hStmt, (SQLCHAR *)sqlbuf, SQL_NTS);
```

---

### ■ データが0件だった場合

何も取得できなかったときは、空配列を返します：

```json
[]
```

この処理は、配列の先頭で `first = 1` のままだった場合などで判断できます。

---

### ■ 実行例

```bash
curl http://localhost/cgi-bin/api/entries.cgi?after_id=100
```

新着エントリだけがJSON形式で返されます。  
JavaScriptからポーリングしたり、SPAアプリから使う際にも便利な形です。

---

### ■ 小まとめ

- クエリパラメータは `getenv("QUERY_STRING")` から取得
- `after_id` を整数に変換してSQLに組み込む
- 新着だけを返すAPIは、クライアントとの連携性を大きく向上させる

このあとは、**JSON出力のエスケープ処理の関数化・再利用**について整理していきます！

## 10.5 JSON出力のエスケープ処理と関数化

C言語でJSON形式の出力を行う場合、文字列中に現れる特殊文字（ダブルクォートやバックスラッシュなど）を  
**正しくエスケープして出力する**必要があります。これを怠ると、出力されたJSONが壊れてしまい、クライアント側でパースエラーになります。

---

### ■ エスケープが必要な文字

| 文字       | JSONでの表現   |
|------------|----------------|
| `"`        | `\"`          |
| `\`       | `\\`         |
| 制御文字   | `\n`, `\r` など |
| バックスペース等 | `\b`, `\f`, `\t` |

---

### ■ エスケープ関数の実装例（簡易版）

以下は、入力文字列をJSON形式に変換する簡易関数です：

```c
void json_escape(const char *src, char *dest, size_t size) {
    size_t i = 0;
    while (*src && i < size - 1) {
        switch (*src) {
            case '"': if (i + 2 < size) { dest[i++] = '\\'; dest[i++] = '"'; } break;
            case '\\': if (i + 2 < size) { dest[i++] = '\\'; dest[i++] = '\\'; } break;
            case '\n': if (i + 2 < size) { dest[i++] = '\\'; dest[i++] = 'n'; } break;
            case '\r': if (i + 2 < size) { dest[i++] = '\\'; dest[i++] = 'r'; } break;
            case '\t': if (i + 2 < size) { dest[i++] = '\\'; dest[i++] = 't'; } break;
            default:
                if ((unsigned char)*src < 0x20) {
                    // 制御文字はスキップまたは \uXXXX に変換すべき（省略）
                } else {
                    dest[i++] = *src;
                }
        }
        src++;
    }
    dest[i] = '\0';
}
```

※ バッファオーバーランを避けるため、`size` のチェックを必ず入れましょう。

---

### ■ 使用例

```c
char raw_body[1024];       // ODBCから取得した内容
char escaped_body[2048];   // エスケープ後の出力用バッファ

strcpy(raw_body, "これは"テスト"です\n改行も含まれます。");
json_escape(raw_body, escaped_body, sizeof(escaped_body));

printf("{ \"body\": \"%s\" }", escaped_body);
```

出力例：

```json
{ "body": "これは\"テスト\"です\n改行も含まれます。" }
```

---

### ■ 関数として共通化する意義

このような関数を `json_utils.c` のようなファイルにまとめておけば、  
以後のAPI実装で毎回エスケープ処理を考える必要がなくなります。

- 再利用性の向上
- 安全性の確保
- 保守性の向上（バグ修正も一箇所でOK）

---

### ■ 小まとめ

- JSONでは特定の文字を必ずエスケープしなければならない
- C言語では `printf()` を使う場合、**自前でエスケープ処理を実装**する必要がある
- 専用の関数に切り出すことで、**他のCGI/APIでも使い回しやすくなる**

次節では、このCGIのApache側設定や補足事項（Appendixとの関連）を解説します。

## 10.6 エラーハンドリングとAPIとしてのふるまい

CGIを「API」として活用する場合、HTMLとは異なる**応答のルール**を意識する必要があります。  
この節では、**JSONでのエラー応答の形式**や**HTTPステータスコードの扱い**、**出力時の注意点**を整理します。

---

### ■ APIに求められる基本的な応答

| 状況             | Content-Type           | HTTPステータス | ボディの形式                       |
|------------------|------------------------|----------------|------------------------------------|
| 正常終了         | application/json       | 200 OK         | `{"status": "success", ...}`       |
| クライアントエラー | application/json       | 400 Bad Request| `{"status": "error", "reason": "〜"}` |
| サーバーエラー   | application/json       | 500 Internal Server Error | `{"status": "error", "reason": "内部エラー"}` |

---

### ■ CGIでのエラー処理方法

CGIでは、HTTPステータスを出力するために、以下のような書き方を使います：

```c
printf("Status: 400 Bad Request\n");
printf("Content-Type: application/json\n\n");
printf("{ \"status\": \"error\", \"reason\": \"after_idが不正です\" }");
```

ただし、`Status:` ヘッダは **必ずContent-Typeより前**に出力する必要があります。

---

### ■ バリデーションエラーの扱い例

```c
int after_id = get_after_id();
if (after_id < 0) {
    printf("Status: 400 Bad Request\n");
    printf("Content-Type: application/json\n\n");
    printf("{ \"status\": \"error\", \"reason\": \"after_idが不正な値です\" }\n");
    return 1;
}
```

---

### ■ サーバー内部エラー時の出力例

```c
if (SQLExecDirect(...) != SQL_SUCCESS) {
    printf("Status: 500 Internal Server Error\n");
    printf("Content-Type: application/json\n\n");
    printf("{ \"status\": \"error\", \"reason\": \"データベースエラー\" }\n");
    return 1;
}
```

※ セキュリティ上の理由から、**内部の詳細（SQL文など）をそのまま返さない**のが望ましいです。

---

### ■ 小まとめ

- APIとしてのCGIは、**ステータスコードとエラーメッセージ**をJSONで明示することが重要
- `Status:` ヘッダの出力順序に注意（最初に出力）
- 内部エラー時には、情報を伏せつつ**クライアントが判定できる応答**を返す

次は、ApacheのMIME設定やファイル拡張子、`.htaccess` による調整など、デプロイに関するTipsを紹介します！

## 10.7 Appendixとの関連（MIME設定・Apache調整）

この章で作成した「JSONを返すCGI」が正しく動作するためには、Webサーバ（Apache）側でもいくつかの調整が必要です。  
この節では、**ApacheのMIME設定やCGIの拡張子の扱い、.htaccessによる局所的設定**など、実運用で重要になるポイントを解説します。

---

### ■ MIMEタイプの設定

Apacheが `.cgi` などの拡張子に対して適切な `Content-Type` を返すように、以下のような設定を行うと良いです：

#### `httpd.conf` または `.htaccess` に追加：

```apache
AddType application/json .json
```

これは `.json` 拡張子の静的ファイルにも効果がありますが、CGIプログラムには直接影響しません。  
CGIが返す `Content-Type` は**プログラム側が制御する**ため、明示的に出力する必要があります。

---

### ■ CGIとして許可される拡張子

`.cgi` 以外にも `.json.cgi` や `.api` を使いたい場合は、`ScriptAlias` や `AddHandler` の調整が必要です：

```apache
AddHandler cgi-script .cgi .api
```

または特定のディレクトリに対して：

```apache
<Directory "/var/www/html/api">
    Options +ExecCGI
    AddHandler cgi-script .api
</Directory>
```

---

### ■ `.htaccess` での局所的な設定

ユーザごとの開発環境では、`.htaccess` を使って個別に設定を追加する方法もあります。

```apache
Options +ExecCGI
AddHandler cgi-script .api
```

この設定を `api/.htaccess` に記述すれば、同ディレクトリ内で `.api` ファイルがCGIとして動作するようになります。

---

### ■ 文字コードの注意点

Apacheが `.json` ファイルに `Content-Type: application/json; charset=UTF-8` を自動付与することがあります。  
CGI側でも、明示的にUTF-8を指定する場合は以下のように書きます：

```c
printf("Content-Type: application/json; charset=UTF-8\n\n");
```

---

### ■ 小まとめ

- CGIはプログラム側で `Content-Type` を出力する必要がある（Apache任せではない）
- 拡張子 `.cgi` 以外を使いたい場合は `AddHandler` 設定が必要
- `.htaccess` を使えばユーザディレクトリでも柔軟に設定できる

次はいよいよ、この章のまとめと応用可能性について触れていきます！

## 10.8 まとめと発展

この章では、C言語CGIで **JSON形式のレスポンスを返すAPI** を構築する方法を学びました。  
シンプルな実装ながら、**フロントエンドと分離された「データの提供手段」**を作れることが確認できたと思います。

---

### ■ 本章で扱ったことのまとめ

- `Content-Type: application/json` を返すことで、HTML以外のデータ応答が可能になる
- ODBC経由で取得したデータを、JSON配列として手動構築する方法
- `after_id` パラメータを利用した「新着フィルタ」の実装
- JSON出力に必要なエスケープ処理と、安全な文字列生成関数の考え方
- `Status:` ヘッダを用いたエラー応答の基本設計
- Apache側での拡張子・MIME設定の調整ポイント

---

### ■ CGIでもAPIはつくれる！

一般に「API」と聞くと、Node.js や Flask、Spring Boot などの近代的なフレームワークを思い浮かべがちです。  
しかし、本章で見てきたように：

- CGIでも十分にAPIは実装可能
- 必要なのは「入出力の構造」と「プロトコルへの理解」
- 軽量で透明性が高く、仕組みが明快

という特性を活かせば、**「仕組みが見えるAPIサーバ」として教育的にも実用的にも活用できます。**

---

### ■ 発展的なトピック

さらに踏み込むと、以下のような発展も可能です：

- クライアント（JavaScript/Ajax）からのAPI呼び出し
- フロントエンド分離構成（SPAとの連携）
- POST/PUT/DELETEによるRESTful API構築
- JSONライブラリ（cJSON など）による構造構築の自動化
- FastCGI化による高速化・プロセス共有

---

### ■ 小まとめ

- C言語CGIでもAPIは作れる！それも実用的に。
- WebページとAPIの境界が理解できると、設計の幅が一気に広がる
- 「画面を出すだけ」で終わらないC言語CGIの世界がここにある

---

> 「APIなんてフレームワークがないと無理」  
> ……そんなこと、なかっただろ？

次章では、Appendixとして開発環境・デプロイ手順などを詳しく解説します！
