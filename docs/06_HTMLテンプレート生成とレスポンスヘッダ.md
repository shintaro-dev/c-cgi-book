# 6. HTMLテンプレート生成とレスポンスヘッダ

前章では、フォーム値をパースして構造化する方法を学びました。  
この章では、ユーザの入力をもとにHTMLレスポンスを動的に生成し、Webページとして返す方法を解説します。

---

## HTMLは「文字列」として出力するだけ

CGIでは、Webサーバに対してまずレスポンスヘッダを出力し、そのあとにHTML本文を出力します。

```c
printf("Content-Type: text/html\n\n");
```

このあとに `printf()` を使ってHTMLを出力すれば、ブラウザで表示可能なWebページとなります。

---

## なぜテンプレート化するのか？

C言語で直接HTMLを `printf()` するのは可能ですが、以下のような問題があります：

- 可読性が低くなる
- HTMLの編集が難しい
- 再利用性が低い

そこで、外部にテンプレートファイル（`.html`）を用意し、動的な部分だけ置換する形式を採用します。

---

## テンプレートの記法と例

テンプレートファイル（例：`template.html`）には以下のような記法でプレースホルダを埋め込みます：

```html
<!DOCTYPE html>
<html>
<head><title>{{title}}</title></head>
<body>
  <h1>{{message}}</h1>
</body>
</html>
```

---

## プレースホルダの置換関数（簡易版）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    const char *key;
    const char *value;
} TemplateItem;

void render_template(const char *filename, TemplateItem *context, int context_size, FILE *out) {
    FILE *fp = fopen(filename, "r");
    if (!fp) {
        fprintf(out, "<p>Template not found: %s</p>", filename);
        return;
    }

    fseek(fp, 0, SEEK_END);
    long size = ftell(fp);
    rewind(fp);

    char *buffer = malloc(size + 1);
    fread(buffer, 1, size, fp);
    buffer[size] = '\0';
    fclose(fp);

    for (int i = 0; i < context_size; i++) {
        char placeholder[64];
        snprintf(placeholder, sizeof(placeholder), "{{%s}}", context[i].key);

        char *pos;
        while ((pos = strstr(buffer, placeholder)) != NULL) {
            size_t before = pos - buffer;
            fprintf(out, "%.*s%s", (int)before, buffer, context[i].value);
            buffer = pos + strlen(placeholder);
        }
    }
    fprintf(out, "%s", buffer);
    free(buffer);
}
```

---

## 使用例

```c
int main(void) {
    TemplateItem context[] = {
        { "title", "Hello Page" },
        { "message", "こんにちは、世界！" }
    };

    printf("Content-Type: text/html\n\n");
    render_template("template.html", context, 2, stdout);
    return 0;
}
```

---

## 小まとめ

- CGIのレスポンスは「ヘッダ＋HTML文字列」で構成されている
- テンプレートを外部ファイルとして持つことで、コードの可読性と再利用性が向上
- プレースホルダを使った置換処理で、C言語でも柔軟にHTMLを生成できる

次章では、実際にデータベースと接続し、フォーム入力を保存・出力する処理に取り組みます。
