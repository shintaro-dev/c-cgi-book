# 7. ODBCを使ったデータベース連携

## 7.1 データベースとCGI

これまでの章では、フォームから受け取ったデータをCGIプログラム内部で処理してきました。  
しかし、実際のWebアプリケーションでは、**データをファイルに保存するだけでは不十分**なことが多くなります。

たとえば…

- ユーザー登録情報を管理したい
- 投稿されたメッセージを一覧表示したい
- 商品一覧をカテゴリごとに絞り込みたい

こういったケースでは、  
**「たくさんのデータを安全に、素早く、柔軟に扱える仕組み」**が必要になります。

これを支えるのが、**データベース（Database）** です。

---

### データベースとは？

データベースとは、  
**大量のデータを効率的に保存・検索・更新できる仕組み**を指します。

特に、Webアプリケーションでよく使われるデータベースには、

- MySQL / MariaDB
- PostgreSQL
- SQLite
- Oracle Database

などがあります。

これらのデータベースとアプリケーションの間をつなぐことで、  
ユーザーがアクセスするたびに「最新のデータ」をリアルタイムにやりとりできるようになります。

---

### なぜCGIとデータベースを連携するのか？

CGIプログラム単体では、**受け取ったデータを一時的に処理する**ことはできます。  
しかし、データベースを使うと、

- データを永続的に保存できる
- 大量のデータから必要なものだけを効率的に取り出せる
- 同時に複数のユーザーが利用できる

といった**本格的なWebアプリケーションに必要な機能**が実現できるようになります。

CGIとデータベースの連携は、  
単なる「問い合わせフォーム」を超えて、**本格的な「Webサービス」へとステップアップするための重要な技術**なのです。

---

### C言語からデータベースにアクセスするには？

世の中には、PHPやPythonのような**データベースアクセス用ライブラリが標準で豊富な言語**もあります。  
しかし、C言語では**直接データベースと通信する機能は標準装備されていません**。

そこで登場するのが、**ODBC（Open Database Connectivity）**という共通仕様です。

ODBCを使うことで、  
**「データベース製品ごとの違いを吸収しながら」**  
**「C言語から共通の手順でデータベースに接続・操作」**できるようになります。

また、C言語には他にも、データベースにアクセスする手段として  
**埋め込みSQL方式**（例えばOracleの**Pro\*C**やInformixの**ESQL/C**など）も存在します。

埋め込みSQL方式では、Cソースコード中に直接SQL文を記述し、  
専用のプリコンパイラによってCソース＋SQLを通常のC言語ソースに変換します。  
この方法は高いパフォーマンスが期待できる一方で、  
**データベース製品に強く依存しやすく、移植性に難がある**というデメリットもあります。

本書では、**製品依存をできるだけ避け、標準的な方法で幅広く対応できる**よう、  
ODBCを採用して進めていきます。

---

### 本章のゴール

この章では、  
C言語のCGIプログラムから**ODBCを使ってデータベースに接続し、データの読み書きを行う**方法を学びます。

また、  
「フォームから受け取ったデータをデータベースに保存する」  
「データベースの内容をHTML形式で一覧表示する」  
といった実践的な処理も体験していきます。

> ※ 本章のサンプルを実行するには、ODBCドライバと接続設定（DSN登録不要）が必要です。  
> （詳しくはAppendix「環境構築ガイド」を参照してください）

それでは、まずはODBCの仕組みから見ていきましょう！

---

## 7.2 ODBCの仕組み

### 7.2.1 ODBCとは何か？

ODBCとは、**Open Database Connectivity（オープン・データベース・コネクティビティ）**の略です。  
その名前の通り、**異なる種類のデータベースに「共通の方法」でアクセスできるようにするための仕組み**です。

たとえば、  
本来ならMySQLとPostgreSQLでは、データベースへの接続手順やコマンドが異なります。  
しかし、ODBCを使うと、アプリケーション側は**共通の関数と命令**だけを使えばよくなります。

> **イメージ図：**  
> - アプリケーション（C言語CGI） → 【ODBC】 → データベース（MySQL, PostgreSQL, など）

つまり、  
**「データベース製品ごとの違いを吸収する中間レイヤー」**  
それがODBCの役割です。

---

### なぜODBCを使うのか？

C言語で直接データベースと通信しようとすると、  
各DB製品専用のライブラリを使わなければなりません。

- MySQL専用のライブラリ（libmysqlclient）
- PostgreSQL専用のライブラリ（libpq）

また、別の方法として  
**埋め込みSQL方式（Pro\*C、ESQL/Cなど）**も存在します。  
この方式では、Cソースコード中に直接SQLを記述し、プリコンパイルしてCコードに変換する手法です。  
ただし、**特定のデータベース製品に依存しやすい**ため、移植性や将来性を考えると課題もあります。

ODBCを使えば、  
アプリケーション側は「ODBC標準仕様」にだけ従えばよいため、  
**将来、データベースを変更する場合も最小限の影響で済む**のです。

---

> ✅ まとめ
> 
> - ODBCは「異なるデータベースに共通手順でアクセスできる仕組み」
> - アプリケーションの移植性と柔軟性を高める
> - C言語からデータベースを扱うときの強力な武器になる

---

次は、  
**「ODBCって具体的にどんな部品でできてるの？」**  
を見ていきましょう！

---

### 7.2.2 ODBCの構成要素

ODBCは、ただ一つの部品でできているわけではありません。  
実際には、次の**三つの要素**が連携して動いています。

---

#### アプリケーション（今回のCGIプログラム）

アプリケーションは、ユーザーのリクエストに応じてデータベースにアクセスする側です。  
私たちがこれから書く**C言語のCGIプログラム**が、まさにこの役割を担います。

アプリケーションは、データベースに直接話しかけるのではなく、  
**ODBCのAPI**（`SQLDriverConnect`や`SQLExecDirect`など）を通して接続・操作を行います。

---

#### ドライバマネージャ

ドライバマネージャは、  
**アプリケーションとドライバの間を仲介する役割**を持っています。

たとえばLinux環境では、`unixODBC`というドライバマネージャがよく使われます。  
Windows環境では、OSに標準で組み込まれています。

ドライバマネージャの役割は：

- アプリケーションからのリクエストを受け取る
- 適切なドライバを選び出して呼び出す
- 必要に応じてエラーメッセージや情報を中継する

といった**交通整理役**です。

アプリケーション側は、**「どのデータベースなのか」**をあまり意識せず、  
共通の手順だけで命令を出せるようになります。

---

#### ドライバ

ドライバは、  
**実際にデータベースと通信を行う本体**です。

たとえばMariaDB用のドライバなら、  
MariaDB特有のプロトコルに従って接続したり、SQLを送ったり、結果を受け取ったりします。

ポイントは、  
**各データベース製品ごとに専用ドライバが存在する**  
ということです。

- MariaDB用 ODBCドライバ
- PostgreSQL用 ODBCドライバ
- SQLite用 ODBCドライバ
などなど。

ドライバを切り替えれば、アプリケーションのコードを大きく変えずに、  
別のデータベースに対応することも可能になります。

> ※ ただし、SQL文の細かい文法（SQL方言）には差異があるため、移植時には注意が必要です。

---

#### 三者の関係図

ここで簡単な流れをまとめておきましょう。

```plaintext
[アプリケーション（CGI）]
          ↓ (SQL発行)
[ドライバマネージャ]
          ↓ (ドライバ選択・中継)
[ドライバ]
          ↓ (DB接続・データ操作)
[データベース（MariaDBなど）]
```

このように、アプリケーションから見れば、  
**ドライバマネージャが適切なドライバを呼び出し、ドライバがDBと直接通信する**  
という二段構えの構成になっています。

---

#### まとめ

- **アプリケーション**はODBC APIを使ってリクエストを出す
- **ドライバマネージャ**はリクエストを中継し、ドライバを管理する
- **ドライバ**は実際にデータベースとやりとりを行う

この構成によって、  
私たちは**データベース製品の違いに振り回されず**に、  
**共通の手順でデータ操作ができる**わけです。

## 7.3 C言語からODBC接続してみよう

これまで、ODBC接続の流れをざっくりと押さえてきました。  
ここからは、いよいよ**実際にコードを書いて接続してみる**ステップに進みます！

まずは**「接続して切断するだけ」**という、最小限のプログラムを作成してみましょう。

---

### 7.3.1 最小限の接続プログラム

以下は、  
**ODBCを使ってデータベースに接続し、すぐに切断するだけ**のシンプルなC言語プログラムです。

```c
#include <stdio.h>
#include <stdlib.h>
#include <sql.h>
#include <sqlext.h>

int main(void)
{
    SQLHENV hEnv = NULL;      // 環境ハンドル
    SQLHDBC hDbc = NULL;      // 接続ハンドル
    SQLRETURN ret;            // 戻り値
    SQLCHAR connStr[] = "DRIVER={MariaDB};SERVER=localhost;DATABASE=testdb;UID=user;PWD=password;";

    // 環境ハンドルを確保
    ret = SQLAllocHandle(SQL_HANDLE_ENV, SQL_NULL_HANDLE, &hEnv);
    if (ret != SQL_SUCCESS && ret != SQL_SUCCESS_WITH_INFO) {
        fprintf(stderr, "環境ハンドルの確保に失敗しました。\n");
        return EXIT_FAILURE;
    }

    // 環境ハンドルの属性設定（ODBC 3.0に設定）
    ret = SQLSetEnvAttr(hEnv, SQL_ATTR_ODBC_VERSION, (void *)SQL_OV_ODBC3, 0);
    if (ret != SQL_SUCCESS && ret != SQL_SUCCESS_WITH_INFO) {
        fprintf(stderr, "環境属性の設定に失敗しました。\n");
        SQLFreeHandle(SQL_HANDLE_ENV, hEnv);
        return EXIT_FAILURE;
    }

    // 接続ハンドルを確保
    ret = SQLAllocHandle(SQL_HANDLE_DBC, hEnv, &hDbc);
    if (ret != SQL_SUCCESS && ret != SQL_SUCCESS_WITH_INFO) {
        fprintf(stderr, "接続ハンドルの確保に失敗しました。\n");
        SQLFreeHandle(SQL_HANDLE_ENV, hEnv);
        return EXIT_FAILURE;
    }

    // データベースに接続
    ret = SQLDriverConnect(
        hDbc, NULL,
        connStr, SQL_NTS,
        NULL, 0, NULL,
        SQL_DRIVER_NOPROMPT
    );

    if (ret == SQL_SUCCESS || ret == SQL_SUCCESS_WITH_INFO) {
        printf("データベースへの接続に成功しました！\n");
    } else {
        fprintf(stderr, "データベースへの接続に失敗しました。\n");
        SQLFreeHandle(SQL_HANDLE_DBC, hDbc);
        SQLFreeHandle(SQL_HANDLE_ENV, hEnv);
        return EXIT_FAILURE;
    }

    // 接続を切断
    SQLDisconnect(hDbc);

    // ハンドルを解放
    SQLFreeHandle(SQL_HANDLE_DBC, hDbc);
    SQLFreeHandle(SQL_HANDLE_ENV, hEnv);

    return EXIT_SUCCESS;
}
```

---

### 7.3.2 コード解説

ポイントだけ簡単に押さえます。

| 手順 | 使った関数 | 説明 |
|:----|:----------|:-----|
| 環境ハンドル確保 | `SQLAllocHandle` | 最初に「ODBC使います！」と宣言 |
| 環境設定 | `SQLSetEnvAttr` | ODBC 3.0準拠に設定 |
| 接続ハンドル確保 | `SQLAllocHandle` | DB接続用のハンドルを確保 |
| データベース接続 | `SQLDriverConnect` | 実際に接続文字列でDBにアクセス |
| 成功／失敗の確認 | 戻り値チェック | 成功なら「成功！」メッセージ、失敗ならエラー |
| 切断・解放 | `SQLDisconnect`、`SQLFreeHandle` | リソースをきれいに片付ける |

---

### 7.3.3 実行例・動作確認

コンパイル：

```bash
gcc -o dbconnect dbconnect.c -lodbc
```

実行：

```bash
./dbconnect
```

結果：

成功時

```
データベースへの接続に成功しました！
```

失敗時

```
データベースへの接続に失敗しました。
```

---

### まとめ

- まずは**「繋がる」ことが最優先**
- この骨格ができれば、次に
  - SQL文の発行
  - データの取得
  に進める
- ODBCは関数の数は多いけれど、**パターン化してしまえば怖くない！**

---
## 7.4 SQLの実行と結果の取得

ODBCでデータベースに接続できるようになったら、  
次はいよいよ**SQL文を実行してデータを操作する**ステップに進みます！

まずは簡単な**INSERT文**を実行して、  
「データベースにレコードを書き込む」動きを体験してみましょう。

---

### 7.4.1 SQL文を実行してみる（INSERT）

以下は、  
**フォームから受け取ったデータを仮定して、データベースにINSERTする**  
サンプルプログラムです。

（今回は簡単化のため、直接プログラム内にデータを埋め込んでいます）

```c
#include <stdio.h>
#include <stdlib.h>
#include <sql.h>
#include <sqlext.h>

int main(void)
{
    SQLHENV hEnv = NULL;
    SQLHDBC hDbc = NULL;
    SQLHSTMT hStmt = NULL;
    SQLRETURN ret;
    SQLCHAR connStr[] = "DRIVER={MariaDB};SERVER=localhost;DATABASE=testdb;UID=user;PWD=password;";
    SQLCHAR insertSql[] = "INSERT INTO sample_table (name, age) VALUES ('Taro', 25)";

    // 環境ハンドル確保
    if (SQLAllocHandle(SQL_HANDLE_ENV, SQL_NULL_HANDLE, &hEnv) != SQL_SUCCESS) {
        fprintf(stderr, "環境ハンドルの確保に失敗しました。\n");
        return EXIT_FAILURE;
    }

    SQLSetEnvAttr(hEnv, SQL_ATTR_ODBC_VERSION, (void *)SQL_OV_ODBC3, 0);

    // 接続ハンドル確保
    if (SQLAllocHandle(SQL_HANDLE_DBC, hEnv, &hDbc) != SQL_SUCCESS) {
        fprintf(stderr, "接続ハンドルの確保に失敗しました。\n");
        SQLFreeHandle(SQL_HANDLE_ENV, hEnv);
        return EXIT_FAILURE;
    }

    // データベース接続
    ret = SQLDriverConnect(hDbc, NULL, connStr, SQL_NTS, NULL, 0, NULL, SQL_DRIVER_NOPROMPT);
    if (ret != SQL_SUCCESS && ret != SQL_SUCCESS_WITH_INFO) {
        fprintf(stderr, "データベース接続に失敗しました。\n");
        SQLFreeHandle(SQL_HANDLE_DBC, hDbc);
        SQLFreeHandle(SQL_HANDLE_ENV, hEnv);
        return EXIT_FAILURE;
    }

    // ステートメントハンドル確保
    if (SQLAllocHandle(SQL_HANDLE_STMT, hDbc, &hStmt) != SQL_SUCCESS) {
        fprintf(stderr, "ステートメントハンドルの確保に失敗しました。\n");
        SQLDisconnect(hDbc);
        SQLFreeHandle(SQL_HANDLE_DBC, hDbc);
        SQLFreeHandle(SQL_HANDLE_ENV, hEnv);
        return EXIT_FAILURE;
    }

    // SQL文実行
    ret = SQLExecDirect(hStmt, insertSql, SQL_NTS);
    if (ret == SQL_SUCCESS || ret == SQL_SUCCESS_WITH_INFO) {
        printf("データのINSERTに成功しました！\n");
    } else {
        fprintf(stderr, "データのINSERTに失敗しました。\n");
    }

    // ステートメントハンドル解放
    SQLFreeHandle(SQL_HANDLE_STMT, hStmt);

    // 接続切断
    SQLDisconnect(hDbc);

    // ハンドル解放
    SQLFreeHandle(SQL_HANDLE_DBC, hDbc);
    SQLFreeHandle(SQL_HANDLE_ENV, hEnv);

    return EXIT_SUCCESS;
}
```

#### このコードのポイント

| 手順 | 関数 | 説明 |
|:----|:-----|:-----|
| ステートメントハンドル確保 | `SQLAllocHandle(SQL_HANDLE_STMT)` | SQL文を実行するためのハンドルを確保 |
| SQL文を発行 | `SQLExecDirect(hStmt, insertSql, SQL_NTS)` | INSERT文をデータベースに送る |
| 実行結果の確認 | 戻り値を見て成功/失敗判定 |
| ステートメントハンドル解放 | `SQLFreeHandle(SQL_HANDLE_STMT)` | 使い終わったら必ず解放 |

---

### 実行例

コンパイル：

```bash
gcc -o insert_test insert_test.c -lodbc
```

実行：

```bash
./insert_test
```

結果：

成功時

```
データのINSERTに成功しました！
```

失敗時

```
データのINSERTに失敗しました。
```

※あらかじめ、`sample_table`テーブルを作成しておく必要があります！

```sql
CREATE TABLE sample_table (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    age INT
);
```

---

### まとめ

- SQL文は `SQLExecDirect` で簡単に実行できる
- ステートメントハンドル（`hStmt`）が新たに登場する
- 実行後はハンドルを必ず解放すること
- 次は、SELECT文を使ってデータを読み取る方法にチャレンジ！

---
### 7.4.2 データを取得する（SELECT）

データベースにINSERTできたら、  
次は**データを取り出す（SELECT）**を体験してみましょう！

今回は、簡単なSELECT文を使って、  
テーブルに登録されているデータを**1行ずつ取得して表示**してみます。

#### サンプルプログラム（SELECT版）

```c
#include <stdio.h>
#include <stdlib.h>
#include <sql.h>
#include <sqlext.h>

int main(void)
{
    SQLHENV hEnv = NULL;
    SQLHDBC hDbc = NULL;
    SQLHSTMT hStmt = NULL;
    SQLRETURN ret;
    SQLCHAR connStr[] = "DRIVER={MariaDB};SERVER=localhost;DATABASE=testdb;UID=user;PWD=password;";
    SQLCHAR selectSql[] = "SELECT id, name, age FROM sample_table";
    SQLINTEGER id;
    SQLCHAR name[100];
    SQLINTEGER age;
    SQLLEN indId, indName, indAge;

    // 環境ハンドル確保
    if (SQLAllocHandle(SQL_HANDLE_ENV, SQL_NULL_HANDLE, &hEnv) != SQL_SUCCESS) {
        fprintf(stderr, "環境ハンドルの確保に失敗しました。\n");
        return EXIT_FAILURE;
    }

    SQLSetEnvAttr(hEnv, SQL_ATTR_ODBC_VERSION, (void *)SQL_OV_ODBC3, 0);

    // 接続ハンドル確保
    if (SQLAllocHandle(SQL_HANDLE_DBC, hEnv, &hDbc) != SQL_SUCCESS) {
        fprintf(stderr, "接続ハンドルの確保に失敗しました。\n");
        SQLFreeHandle(SQL_HANDLE_ENV, hEnv);
        return EXIT_FAILURE;
    }

    // データベース接続
    ret = SQLDriverConnect(hDbc, NULL, connStr, SQL_NTS, NULL, 0, NULL, SQL_DRIVER_NOPROMPT);
    if (ret != SQL_SUCCESS && ret != SQL_SUCCESS_WITH_INFO) {
        fprintf(stderr, "データベース接続に失敗しました。\n");
        SQLFreeHandle(SQL_HANDLE_DBC, hDbc);
        SQLFreeHandle(SQL_HANDLE_ENV, hEnv);
        return EXIT_FAILURE;
    }

    // ステートメントハンドル確保
    if (SQLAllocHandle(SQL_HANDLE_STMT, hDbc, &hStmt) != SQL_SUCCESS) {
        fprintf(stderr, "ステートメントハンドルの確保に失敗しました。\n");
        SQLDisconnect(hDbc);
        SQLFreeHandle(SQL_HANDLE_DBC, hDbc);
        SQLFreeHandle(SQL_HANDLE_ENV, hEnv);
        return EXIT_FAILURE;
    }

    // SQL文実行（SELECT）
    ret = SQLExecDirect(hStmt, selectSql, SQL_NTS);
    if (ret != SQL_SUCCESS && ret != SQL_SUCCESS_WITH_INFO) {
        fprintf(stderr, "SELECT文の実行に失敗しました。\n");
        SQLFreeHandle(SQL_HANDLE_STMT, hStmt);
        SQLDisconnect(hDbc);
        SQLFreeHandle(SQL_HANDLE_DBC, hDbc);
        SQLFreeHandle(SQL_HANDLE_ENV, hEnv);
        return EXIT_FAILURE;
    }

    // データ取得・表示
    printf("id\tname\tage\n");
    printf("--------------------------\n");

    while ((ret = SQLFetch(hStmt)) != SQL_NO_DATA) {
        SQLGetData(hStmt, 1, SQL_C_SLONG, &id, 0, &indId);
        SQLGetData(hStmt, 2, SQL_C_CHAR, name, sizeof(name), &indName);
        SQLGetData(hStmt, 3, SQL_C_SLONG, &age, 0, &indAge);

        printf("%d\t%s\t%d\n", id, name, age);
    }

    // ステートメントハンドル解放
    SQLFreeHandle(SQL_HANDLE_STMT, hStmt);

    // 接続切断
    SQLDisconnect(hDbc);

    // ハンドル解放
    SQLFreeHandle(SQL_HANDLE_DBC, hDbc);
    SQLFreeHandle(SQL_HANDLE_ENV, hEnv);

    return EXIT_SUCCESS;
}
```

#### このコードのポイント

| 手順 | 関数 | 説明 |
|:----|:-----|:-----|
| SQL文発行 | `SQLExecDirect` | SELECT文を実行 |
| 1行ずつ結果取得 | `SQLFetch` | 1レコードずつ取得する |
| 列データ取得 | `SQLGetData` | 指定した列の値を取得する |
| 結果の出力 | `printf` | 取得したデータを表示 |

---

#### 実行例

コンパイル：

```bash
gcc -o select_test select_test.c -lodbc
```

実行：

```bash
./select_test
```

出力結果（例）：

```
id      name    age
--------------------------
1       Taro    25
2       Hanako  30
```

※ あらかじめ、`sample_table`にデータが入っている必要があります。

---

#### まとめ

- SELECT文も `SQLExecDirect` で実行できる
- 結果を受け取るには `SQLFetch` ＋ `SQLGetData` を組み合わせる
- データ型ごとに適切な型で受け取る（`SQL_C_SLONG`, `SQL_C_CHAR`など）

---
### 7.4.3 エラー処理の基本

ここまでで、  
**SQLを実行してデータを追加したり取得したりする方法**を体験しました。

しかし、実際の開発現場では、  
**「失敗するかもしれない」**ことを前提に作らないといけません。

たとえば：

- データベース接続に失敗したら？
- SQL文に誤りがあったら？
- 取得対象のデータが存在しなかったら？

そんなときに、**適切にエラーメッセージを出したり、安全にリソースを解放したりする**のが、  
**エラー処理**の基本になります。

---

#### 最小限のエラー処理パターン

ODBCでは、  
**ほとんどすべての関数が「戻り値」で成功・失敗を返してきます**。

戻り値は、次のように解釈します：

| 戻り値 | 意味 |
|:---|:---|
| `SQL_SUCCESS` | 正常終了 |
| `SQL_SUCCESS_WITH_INFO` | 正常終了だけど追加情報あり（警告） |
| `SQL_NO_DATA` | データなし（特にSELECT時に使う） |
| `SQL_ERROR` | エラー発生 |
| `SQL_INVALID_HANDLE` | ハンドルが無効 |

---

#### 基本的なチェック方法

実際のコードでは、たとえばこんな風に書きます。

```c
SQLRETURN ret;

ret = SQLExecDirect(hStmt, sql, SQL_NTS);
if (ret != SQL_SUCCESS && ret != SQL_SUCCESS_WITH_INFO) {
    fprintf(stderr, "SQL文の実行に失敗しました。\n");
    // 必要ならここでリソース解放やエラー復帰処理を入れる
}
```

- **成功 (`SQL_SUCCESS` or `SQL_SUCCESS_WITH_INFO`) なら続行**
- **それ以外 (`SQL_ERROR`など) ならエラー処理に入る**

このパターンを覚えておくだけで、  
ODBCプログラムの安全性がグッと高まります。

---

#### エラー情報を詳しく取得するには？

さらに踏み込んで、  
**エラーの内容（原因）を取得する**こともできます。

ODBCでは、`SQLGetDiagRec` という関数を使って、  
エラーコードやメッセージを取得できます。

簡単な例：

```c
SQLCHAR sqlState[6];
SQLINTEGER nativeError;
SQLCHAR messageText[256];
SQLSMALLINT textLength;

if (ret == SQL_ERROR) {
    SQLGetDiagRec(SQL_HANDLE_STMT, hStmt, 1, sqlState, &nativeError, messageText, sizeof(messageText), &textLength);
    fprintf(stderr, "ODBCエラー: %s (%s)\n", messageText, sqlState);
}
```

これで、  
**「何が起きたのか？」**をログに記録できるようになります。

---

#### まとめ

- ODBC関数の**戻り値を必ずチェックする**
- `SQL_SUCCESS`か`SQL_SUCCESS_WITH_INFO`ならOK
- エラーが出たら最低限のメッセージを出す
- 必要に応じて`SQLGetDiagRec`で詳細エラーを取得できる

---
