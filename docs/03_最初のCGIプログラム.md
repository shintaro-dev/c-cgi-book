# 最初のCGIプログラム

## Webブラウザに「Hello, CGI」と表示してみよう

ここでは最小限のCプログラムを書いて、ブラウザに文字を表示してみます。  
これによって、**CGIが「HTMLを返すプログラム」だという感覚**が体験できます。

---

## CGIプログラムの要件と配置

CGIプログラムは、通常 Webサーバ（Apacheなど）に設定された `cgi-bin` ディレクトリに置く必要があります。  
また、実行可能ファイルでなければなりません。

```bash
# 例：hello.cgi を cgi-bin に配置して実行権限を付ける
chmod +x hello.cgi
```

CGIは単なるスクリプトではなく、**「外部プログラム」**としてWebサーバから呼び出されることを忘れずに。

---

## Cで書く最小のCGIプログラム

以下は「Hello, CGI」と表示する最小構成のC言語CGIプログラムです。

```c
#include <stdio.h>

int main(void) {
    // HTTPレスポンスヘッダ
    printf("Content-Type: text/html\n\n");

    // HTML本文
    printf("<!DOCTYPE html>\n");
    printf("<html><head><title>Hello</title></head><body>\n");
    printf("<h1>Hello, CGI</h1>\n");
    printf("</body></html>\n");

    return 0;
}
```

これを `hello.c` として保存し、以下のようにビルドします：

```bash
gcc hello.c -o hello.cgi
```

生成された `hello.cgi` をWebサーバの `cgi-bin` に配置すれば、  
ブラウザから `http://localhost/cgi-bin/hello.cgi` にアクセスすることで動作を確認できます。

---

## うまく表示されないときは？

- 実行権限があるか？ → `chmod +x hello.cgi`
- Webサーバの設定が正しいか？（`ScriptAlias` ディレクティブなど）
- `Content-Type` ヘッダを忘れていないか？

---

## 実行ファイルのオーナーとパーミッション

CGIプログラムは、Webサーバが“外部から呼び出す実行ファイル”として動作します。  
そのため、次の点に気をつけましょう：

- ファイルの所有者がApacheの実行ユーザ（例：`www-data`）である必要はありません
- ただし、**実行権限（x）がothersにもある必要があります**（例：`chmod 755 hello.cgi`）

パーミッションが不適切な場合は、ブラウザ上で `500 Internal Server Error` が返ってくることがあります。

---

## 補足：ユーザディレクトリでCGIを動かすには？

Apacheの設定によっては、`/usr/lib/cgi-bin` のようなシステム全体のディレクトリではなく、  
ユーザごとの `~/public_html` 以下でCGIを実行できることがあります。

この機能は **mod_userdir** モジュールによって提供されており、  
URLとしては以下のようになります：

```
http://localhost/~ユーザ名/cgi-bin/hello.cgi
```

ただし、**この機能はデフォルトで無効になっていることも多いため、使用するには以下の準備が必要です：

- Apacheの `mod_userdir` モジュールを有効にする（`a2enmod userdir`）
- 該当ユーザのホームディレクトリに `public_html/cgi-bin` を作成
- ディレクトリに適切なパーミッションを設定（例：701以上）
- Apacheの設定で `ExecCGI` を許可する

初心者には少し難しい構成かもしれませんが、  
**環境を分離して実験したい場合や、root権限が使えない開発環境では有効な方法です。**

---

次章では、フォームからの入力を受け取り、CGIプログラムがデータをどう受け取るかを体験します。
