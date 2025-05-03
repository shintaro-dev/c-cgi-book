## 実践アプリ：簡易CMS（日記投稿アプリ）を作ろう

### 9.1 アプリの概要と要件整理

> ※ 本章では日記アプリの投稿・保存・表示の仕組みを明確に理解することを優先し、HTMLテンプレート処理は使用していません。
> 各ページのHTMLは直接出力していますが、6章で紹介したテンプレート処理を用いることで、保守性・再利用性を高めることができます。
> Appendixにてテンプレート対応版の例も紹介予定です。

ここからは、実践的なアプリケーション開発に挑戦します！

今回作るのは、**「簡易CMS（日記投稿アプリ）」**です。

---

### アプリの概要

ログインユーザーが「日記」を投稿・管理できるシンプルなWebアプリケーションを作成します。  
投稿はデータベースに保存され、ログインユーザが投稿した日記の公開・非公開を設定出来ます。  
公開された投稿は他のユーザーにも閲覧可能ですが、非公開投稿は投稿者本人のみ閲覧できます。

---

### 機能要件

- **ユーザー認証**  
  ログイン済みのユーザーのみ、日記の投稿・閲覧ができる。

- **日記投稿**  
  ユーザーは日記（タイトル・本文）を入力し、投稿できる。

- **公開設定**  
  投稿時に「公開」「非公開」の設定を選択できる。

- **データベース保存**  
  投稿データ（ユーザーID、タイトル、本文、公開設定、投稿日時）はデータベースに保存される。

- **日記一覧表示**  
  - 公開投稿は、すべてのユーザーが閲覧できる。  
  - 非公開投稿は、投稿者本人のみが閲覧できる。

- **ユーザーごとの投稿管理**  
  - 各ユーザーは、自分の投稿を区別して閲覧・管理できる。  
  - 将来的に「編集・削除」機能に拡張可能な構成とする。

---

### データベース設計（シンプル版）

仮に**日記データ**を保存するためのテーブルを用意します。

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

- `user_id`：投稿者のID（セッションと紐づく）
- `title`：日記のタイトル
- `body`：日記本文
- `is_public`：公開（1）・非公開（0）フラグ
- `created_at`：投稿日時

---

### 進め方の方針

まずは、

- 誰でも投稿できるシンプル版（ログイン不要）
- 公開/非公開切り替え版

を作成し、

その後、

- ログイン機能との連携版（ユーザーごとに管理）

へ段階的に発展させていきます！

> ※ 本章ではまず「動くものを作る」ことを最優先に進め、細かなセキュリティ対策やUI改善は後続章で扱います。

---

次は、
**「日記投稿フォームを作成する」**パートに進みましょう！

### 9.2 日記投稿フォームを作成する

ここでは、ユーザーが日記を投稿できるように、
HTMLフォームを作成します。

まずは必要最小限のシンプルなフォームを作りましょう！

#### フォーム設計方針

- タイトル（1行テキスト）
- 本文（複数行テキストエリア）
- 公開/非公開選択（ラジオボタン）
- 送信ボタン

---

#### 投稿フォームのサンプルコード

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>日記投稿フォーム</title>
</head>
<body>
    <h1>日記を投稿する</h1>
    <form action="submit_diary.cgi" method="post">
        <p>
            <label>タイトル：<br>
            <input type="text" name="title" size="40" required></label>
        </p>
        <p>
            <label>本文：<br>
            <textarea name="body" rows="10" cols="60" required></textarea></label>
        </p>
        <p>
            <label>公開設定：<br>
            <input type="radio" name="is_public" value="1" checked> 公開
            <input type="radio" name="is_public" value="0"> 非公開</label>
        </p>
        <p>
            <input type="submit" value="投稿する">
        </p>
    </form>
</body>
</html>
```

---

#### ポイント解説

| 項目 | 説明 |
|:---|:---|
| フォームの送信先 | `submit_diary.cgi` にPOST送信する設計 |
| タイトル | 必須入力、最大40文字程度を想定 |
| 本文 | 必須入力、複数行入力可能なエリア |
| 公開設定 | ラジオボタンで「公開」か「非公開」を選択 |

> ※ バリデーション（入力チェック）はHTMLレベルで`required`指定していますが、
>  サーバ側でもしっかりチェックする設計にします（後述）。

---

次は、
**「フォーム送信を受け取るCGIプログラムを作成する」**パートに進みましょう！

### 9.3 フォームデータを受信して処理する

ここでは、投稿フォームから送信されたデータを受け取り、
必要な処理を行うCGIプログラム（`submit_diary.cgi`）を作成します。

#### 9.3.1 データ受信と基本チェック

まずは、CGIプログラムでPOSTデータを受け取って、
必須項目（タイトル・本文・公開設定）が正しく送信されているか確認します。

#### サンプルコード（C言語 CGI）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MAX_INPUT 4096

int main(void)
{
    char input[MAX_INPUT];
    char *title = NULL, *body = NULL, *is_public = NULL;

    // HTTPヘッダ出力
    printf("Content-Type: text/html\r\n\r\n");

    // POSTデータ受信
    if (fgets(input, sizeof(input), stdin) == NULL) {
        printf("<p>データ受信に失敗しました。</p>");
        return 1;
    }

    // タイトル抽出（簡易版）
    title = strstr(input, "title=");
    body = strstr(input, "body=");
    is_public = strstr(input, "is_public=");

    if (!title || !body || !is_public) {
        printf("<p>必須項目が不足しています。</p>");
        return 1;
    }

    printf("<h1>受信成功</h1>");
    printf("<p>データを受信しました。ここからデータベース保存処理に進みます。</p>");

    return 0;
}
```

---

#### ポイント解説

| 項目 | 説明 |
|:---|:---|
| POSTデータ受信 | `stdin`から一括で読み込む（本格運用時はサイズチェックや分割受信が必要） |
| データ存在チェック | タイトル・本文・公開設定の有無を最低限チェック |
| 出力 | 必須エラー時はエラーメッセージ表示 |

> ※ 本サンプルでは超簡易版です。
> 本格的にはフォーム値のエンコード解析（URLデコード）も必要ですが、
> 詳細は後続章で解説します！

---

次は、
**「受信データをデータベースに保存する処理」を作成する**パートに進みます！

### 9.4 データベースに保存する

ここでは、フォームから受信したデータを、ODBCを使ってデータベースに保存する処理を実装します。

#### 9.4.1 必要な準備

- ODBCドライバと接続設定が済んでいること（Appendix参照）
- データベースに `diary` テーブルが作成済みであること

---

#### 9.4.2 サンプルコード（C言語＋ODBC）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sql.h>
#include <sqlext.h>

// 入力を安全に処理するための関数（簡易エスケープ）
void escape(char *dest, const char *src, size_t max) {
    size_t i = 0, j = 0;
    while (src[i] && j < max - 1) {
        if (src[i] == '\'' || src[i] == '"' || src[i] == '\\') {
            if (j < max - 2) dest[j++] = '\\';
        }
        dest[j++] = src[i++];
    }
    dest[j] = '\0';
}

int main(void)
{
    SQLHENV env;
    SQLHDBC dbc;
    SQLHSTMT stmt;
    SQLRETURN ret;
    char *title_raw = getenv("title");
    char *body_raw = getenv("body");
    char *pub_raw = getenv("is_public");

    char title[1024], body[4096];
    escape(title, title_raw ? title_raw : "", sizeof(title));
    escape(body, body_raw ? body_raw : "", sizeof(body));
    int is_public = (pub_raw && strcmp(pub_raw, "0") == 0) ? 0 : 1;

    // HTTPレスポンスヘッダ
    printf("Content-Type: text/html\r\n\r\n");

    // ODBC初期化
    ret = SQLAllocHandle(SQL_HANDLE_ENV, SQL_NULL_HANDLE, &env);
    SQLSetEnvAttr(env, SQL_ATTR_ODBC_VERSION, (void *)SQL_OV_ODBC3, 0);
    SQLAllocHandle(SQL_HANDLE_DBC, env, &dbc);

    // DSNなし接続（例：MariaDB）
    SQLCHAR connStr[] = "DRIVER={MariaDB};SERVER=localhost;DATABASE=testdb;UID=user;PWD=password;";
    ret = SQLDriverConnect(dbc, NULL, connStr, SQL_NTS, NULL, 0, NULL, SQL_DRIVER_NOPROMPT);

    if (ret != SQL_SUCCESS && ret != SQL_SUCCESS_WITH_INFO) {
        printf("<p>データベース接続に失敗しました。</p>");
        return 1;
    }

    SQLAllocHandle(SQL_HANDLE_STMT, dbc, &stmt);

    char query[8192];
    snprintf(query, sizeof(query),
        "INSERT INTO diary (user_id, title, body, is_public) VALUES (0, '%s', '%s', %d)",
        title, body, is_public);

    ret = SQLExecDirect(stmt, (SQLCHAR *)query, SQL_NTS);

    if (ret == SQL_SUCCESS || ret == SQL_SUCCESS_WITH_INFO) {
        printf("<p>投稿が保存されました！</p>");
    } else {
        printf("<p>保存中にエラーが発生しました。</p>");
    }

    SQLFreeHandle(SQL_HANDLE_STMT, stmt);
    SQLDisconnect(dbc);
    SQLFreeHandle(SQL_HANDLE_DBC, dbc);
    SQLFreeHandle(SQL_HANDLE_ENV, env);

    return 0;
}
```

---

#### 9.4.3 解説ポイント

| 項目 | 説明 |
|:---|:---|
| エスケープ処理 | SQLインジェクション対策の簡易対応（本格版はプリペアドステートメント推奨） |
| `getenv()` | 環境変数からCGIパラメータ取得（Apacheの設定により形式は異なる） |
| SQL組み立て | 文字列連結によるSQL生成（デバッグしやすい） |

> ※ 本番環境では、SQLインジェクション対策として `SQLBindParameter` 等を用いたプリペアドステートメントを推奨します。

---

次は、  
**「投稿された日記を一覧表示する」**パートに進みましょう！

### 9.5 日記を一覧表示する

ここでは、データベースに保存された日記を一覧表示する機能を作成します。

#### 9.5.1 一覧表示の要件

- 投稿された日記の一覧をHTML形式で表示する
- 公開設定（is_public）に応じて表示制御を行う
- 表示順は新しい投稿が上に来るようにする（ORDER BY created_at DESC）

---

#### 9.5.2 サンプルコード（C言語＋ODBC）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sql.h>
#include <sqlext.h>

int main(void)
{
    SQLHENV env;
    SQLHDBC dbc;
    SQLHSTMT stmt;
    SQLRETURN ret;
    SQLCHAR title[256], body[2048];

    // HTTPヘッダ
    printf("Content-Type: text/html\r\n\r\n");
    printf("<html><head><meta charset='UTF-8'><title>日記一覧</title></head><body>");
    printf("<h1>公開日記一覧</h1>");

    // ODBC接続準備
    SQLAllocHandle(SQL_HANDLE_ENV, SQL_NULL_HANDLE, &env);
    SQLSetEnvAttr(env, SQL_ATTR_ODBC_VERSION, (void *)SQL_OV_ODBC3, 0);
    SQLAllocHandle(SQL_HANDLE_DBC, env, &dbc);

    SQLCHAR connStr[] = "DRIVER={MariaDB};SERVER=localhost;DATABASE=testdb;UID=user;PWD=password;";
    ret = SQLDriverConnect(dbc, NULL, connStr, SQL_NTS, NULL, 0, NULL, SQL_DRIVER_NOPROMPT);

    if (ret != SQL_SUCCESS && ret != SQL_SUCCESS_WITH_INFO) {
        printf("<p>データベース接続に失敗しました。</p></body></html>");
        return 1;
    }

    SQLAllocHandle(SQL_HANDLE_STMT, dbc, &stmt);

    const char *sql = "SELECT title, body FROM diary WHERE is_public = 1 ORDER BY created_at DESC";
    SQLExecDirect(stmt, (SQLCHAR *)sql, SQL_NTS);

    while ((ret = SQLFetch(stmt)) != SQL_NO_DATA) {
        SQLGetData(stmt, 1, SQL_C_CHAR, title, sizeof(title), NULL);
        SQLGetData(stmt, 2, SQL_C_CHAR, body, sizeof(body), NULL);

        printf("<div style='border:1px solid #ccc; padding:10px; margin:10px;'>");
        printf("<h2>%s</h2>\n", title);
        printf("<p>%s</p>\n", body);
        printf("</div>\n");
    }

    SQLFreeHandle(SQL_HANDLE_STMT, stmt);
    SQLDisconnect(dbc);
    SQLFreeHandle(SQL_HANDLE_DBC, dbc);
    SQLFreeHandle(SQL_HANDLE_ENV, env);

    printf("</body></html>");
    return 0;
}
```

---

#### 9.5.3 解説ポイント

| 項目 | 説明 |
|:---|:---|
| `ORDER BY created_at DESC` | 新しい投稿が上に来るように並び替え |
| `WHERE is_public = 1` | 公開されている投稿のみを取得 |
| `SQLGetData` | 1件ずつタイトル・本文を取り出して表示 |

> ※ 本サンプルでは「投稿者ごとの制御」や「非公開投稿の表示制限」は実装していません。
> 後続のログイン機能と組み合わせて発展させていきます。

---

これで、
**投稿 → 保存 → 一覧表示** の基本的な一連の流れが完成しました！

次は、
**「ログインユーザーごとの投稿表示」**など、ユーザー管理の話題に進んでいきます。

### 9.6 ログインユーザーごとの投稿制御

この節では、ログインしているユーザーのセッション情報を使って、
「誰が投稿したか」「誰に見せるか」を制御する仕組みを構築します。

---

#### 9.6.1 投稿者情報の保存

投稿時にセッションからユーザーIDを取得し、
データベースの `user_id` カラムに保存するようにします。

```c
// セッションIDからユーザーIDを取得する関数（仮）
int get_user_id_from_session() {
    // 例：SESSION_ID をキーに /tmp/session/[ID] を読み、ユーザーIDを返す
    // 実際の実装は8章のセッション処理に依存
    return 42; // 仮のユーザーID（後で置き換える）
}

...

int user_id = get_user_id_from_session();

snprintf(query, sizeof(query),
    "INSERT INTO diary (user_id, title, body, is_public) VALUES (%d, '%s', '%s', %d)",
    user_id, title, body, is_public);
```

---

#### 9.6.2 一覧表示のフィルタ制御

表示対象は次の条件で絞り込みます：

- 公開投稿（`is_public = 1`）
- または、ログイン中のユーザーの投稿（`user_id = セッションIDに紐づくID`）

```sql
SELECT title, body FROM diary
WHERE is_public = 1 OR user_id = [ログインユーザーID]
ORDER BY created_at DESC
```

---

#### 9.6.3 ログイン状態のチェック

一覧ページや投稿ページでは、セッションの有無によって表示や挙動を切り替えます。

- 未ログインの場合は「自分の投稿」は表示されない
- 投稿ページにアクセスしても、ログインしていなければリダイレクトさせる

> この処理は8章の `is_logged_in()` などと連携させることで実現できます。

---

### まとめ

- 投稿時に `user_id` を保存することで「誰の投稿か」を明確にできる
- 一覧表示時に「公開投稿 + 自分の投稿」のみ表示することでアクセス制御が可能になる
- セッション管理を組み合わせることで、本格的な「ユーザー別表示」が実現できる

---

次は、  
**「ログアウト処理の補足」や「セッションセキュリティ強化」の話題に進みます！

### 9.7 ログアウト処理とセッションの後始末

ここでは、ログインセッションを終了するための処理（ログアウト処理）について、
基本の復習と実運用上の注意点を補足します。

---

#### 9.7.1 ログアウト処理の基本フロー

1. セッションID（SESSION_ID）をクッキーから取得
2. 対応するセッションファイル（例：`/tmp/session/xxxxxx`）を削除（`unlink()`）
3. `Set-Cookie` ヘッダで SESSION_ID を無効化（過去日時で上書き）

```c
// クッキーからSESSION_ID取得 → セッションファイル削除例
char *session_id = cgi_get_cookie("SESSION_ID");
if (session_id) {
    char path[256];
    snprintf(path, sizeof(path), "/tmp/session/%s", session_id);
    unlink(path); // セッションファイル削除
    printf("Set-Cookie: SESSION_ID=deleted; Path=/; Expires=Thu, 01 Jan 1970 00:00:00 GMT\n");
}
```

---

#### 9.7.2 セッション削除時の注意点

- セッションファイルが存在しない場合もある（例：タイムアウトや手動削除済）
  → `unlink()` の戻り値は確認し、エラー処理してもよい

- パーミッションエラーで削除できない場合がある
  → CGIが実行されるユーザーとファイル所有者に注意（例：`www-data` vs `root`）

- セッション破棄後は「ログアウト完了画面」か「ログインページにリダイレクト」

---

#### 9.7.3 今後の改善ポイント

> 実運用を想定すると、セッション処理にはさらなる強化が望まれます。

- **セッション有効期限の導入**  
  最終アクセス時刻を記録し、一定時間経過でセッション無効化

- **クッキーのSecure属性・SameSite属性の指定**  
  HTTPS通信限定やクロスサイト制御を強化（例：`Secure; SameSite=Lax`）

- **IPアドレスやUser-Agentによる検証**  
  セッションIDの乗っ取り対策として有効（ただし実装コスト高）

> ※ Appendixで「セッションセキュリティ強化」の補足も予定しています！

---

### まとめ

- セッションファイルを削除し、クッキーを無効化するのが基本のログアウト処理
- `unlink()` の結果やファイルの存在有無を考慮した実装が理想
- 実運用では、セッションの有効期限・セキュリティ属性の強化が重要になる

---

これで、
**ログイン・投稿・表示・ログアウト**の一連のユーザー動作が完結しました！

次はいよいよ、
**Appendix（補足・発展的内容）**に進みます！

> ※ 本章で使用する `diary` テーブルの定義や初期データの投入手順は、  
> 第11章「Appendix：開発環境・デプロイTips集」にも掲載しています。

> ※ テンプレートファイルの配置や文字コード、パーミッションについての注意点は、  
> Appendixにも補足があります。

※ 本章では `html_escape()` を自作していますが、より完全なエスケープ処理を行うには  
既存のライブラリやフレームワークを使用するのが一般的です。