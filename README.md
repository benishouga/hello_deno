# deno 使ってみる会

公式の Getting Started にならって試していきます。
（2.2 の **Using TypeScript** のあたりまでの内容です。）

https://deno.land/manual/getting_started

## インストール

```shell
$ curl -fsSL https://deno.land/x/install/install.sh | sh

Archive:  /home/xxxxx/.deno/bin/deno.zip
  inflating: deno
Deno was installed successfully to /home/xxxxx/.deno/bin/deno
Manually add the directory to your $HOME/.bash_profile (or similar)
  export DENO_INSTALL="/home/xxxxx/.deno"
  export PATH="$DENO_INSTALL/bin:$PATH"
Run '/home/xxxxx/.deno/bin/deno --help' to get started
```

### コメントに従って .bash_profile なり .bashrc に環境変数を追加

```shell
export DENO_INSTALL="/home/xxxxx/.deno"
export PATH="$DENO_INSTALL/bin:$PATH"
```

### インストールできているか確認

```shell
$ deno --version
deno 1.0.2
v8 8.4.300
typescript 3.9.2
```

## IDE サポート

公式で VSCode 用が提供されていた。<br>
https://marketplace.visualstudio.com/items?itemName=denoland.vscode-deno

`import xxx from "https://deno.land/std/http/server.ts";` のような deno 独特な from 指定の型を解決してくれる様子？

JetBrains IDEs や Vim and NeoVim 用も一応あるっぽい。

## Runtime API 型定義

`Deno.xxx` な Runtime API を VSCode 上でも解決できるようにする

Runtime API の型定義を deno のコマンドで出力する

```
$ deno types > deno.d.ts
```

tsconfig.json を作って上記の d.ts を読むようにする

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "deno": ["deno.d.ts"]
    }
  }
}
```

これで `Deno.xxx` な API が VSCode 上でもエラーにならないはず。

ダメな場合は 一度 VSCode 再起動したら良くなったりするかも。

## コードを実行してみる

### ただ走らせる（コンソールログを出してみる）

hello_deno.ts を作る

```typescript
console.log("Welcome to Deno 🦕");
```

```shell
$ deno run hello_deno.ts
Compile file:///home/xxxxx/work/deno/first.ts
Welcome to Deno 🦕
```

ちゃんと走る様子。

### fetch はそのまま使える様子

hello_fetch.ts を作る

```typescript
const url = Deno.args[0];
const res = await fetch(url);

const body = new Uint8Array(await res.arrayBuffer());
await Deno.stdout.write(body);
```

fetch を動かすためには `--alow-net` フラグが必要

```shell
$ deno run --allow-net=example.com hello_fetch.ts https://example.com
<!doctype html>
<html>・・・・・</html>
```

つけないと以下のように permission エラーになる

```
$ deno run hello_fetch.ts https://example.com
error: Uncaught PermissionDenied: network access to "https://example.com/", run again with the --allow-net flag
    at unwrapResponse ($deno$/ops/dispatch_json.ts:43:11)
    at Object.sendAsync ($deno$/ops/dispatch_json.ts:98:10)
    at async fetch ($deno$/web/fetch.ts:591:27)
    at async file:///home/xxxxx/work/deno/hello_fetch.ts:2:13
```

`--allow-net=example.com` の `=example.com` は white list で、指定しないとすべての URL へのアクセスを許可することになる。

```shell
$ deno run --allow-net hello_fetch.ts https://example.com
<!doctype html>
<html>・・・・・</html>
```

`--allow-net=example.com` と指定したときは `example.com` 以外 (以下は `example.net` への例) へのアクセスは permission error になる。

```shell
$ deno run --allow-net=example.com hello_fetch.ts https://example.net
error: Uncaught PermissionDenied: network access to "https://example.net/", run again with the --allow-net flag
    at unwrapResponse ($deno$/ops/dispatch_json.ts:43:11)
    at Object.sendAsync ($deno$/ops/dispatch_json.ts:98:10)
    at async fetch ($deno$/web/fetch.ts:591:27)
    at async file:///home/xxxxx/work/deno/hello_fetch.ts:2:13
```

### ファイルを読み込んで見る

hello_file.ts を作る

```typescript
const filenames = Deno.args;
for (const filename of filenames) {
  const file = await Deno.open(filename);
  await Deno.copy(file, Deno.stdout);
  file.close();
}
```

hoge.txt を適当な内容で作って以下のように実行

```
$ deno run --allow-read hello_file.ts ./hoge.txt
hogehoge-
```

### TCP サーバー立ててみる

hello_tcp.ts を作る

```typescript
const hostname = "0.0.0.0";
const port = 8080;
const listener = Deno.listen({ hostname, port });
console.log(`Listening on ${hostname}:${port}`);
for await (const conn of listener) {
  Deno.copy(conn, conn);
}
```

起動して

```
$ deno run --allow-net hello_tcp.ts
Compile file:///home/xxxxx/work/deno/hello_tcp.ts
Listening on 0.0.0.0:8080
```

別のターミナルから nc 使ってデータを送ってみると、同じ内容でレスポンスを返してくる

```
$ nc localhost 8080
test
test
hoge
hoge
```

### HTTP サーバーを立ててみる

hello_server.ts を作る

```typescript
import { serve } from "https://deno.land/std/http/server.ts";
const server = serve({ port: 8000 });
console.log("http://localhost:8000/");
for await (const req of server) {
  req.respond({ body: "Hello World\n" });
}
```

deno の std ライブラリは上記のように `import ... from` で URL を指定して利用する。

VSCode では URL を最初解釈できないためエディタ上はコンパイルエラーと表示されるが、実行したら実際にモジュールのダウンロードが行われるため、コンパイルエラーとはならずに実行できる。

実行は以下の感じ

```
$ deno run --allow-net hello_server.ts
Download https://deno.land/std/http/server.ts
Warning Implicitly using master branch https://deno.land/std/http/server.ts
Download https://deno.land/std/encoding/utf8.ts
Download https://deno.land/std/io/bufio.ts

・・・・略・・・・

Warning Implicitly using master branch https://deno.land/std/path/interface.ts
Warning Implicitly using master branch https://deno.land/std/path/_globrex.ts
Compile file:///home/xxxxx/work/deno/hello_server.ts
http://localhost:8000/
```

http://localhost:8000/ をブラウザで開くと `Hello World` と表示される。

一度実行すると deno がライブラリをキャッシュするが、VSCode からもそのキャッシュを見に行くようになるため、VSCode 上もコンパイルエラーは出なくなる。

## 感想

- 良い感じなのかどうかはあまり分からず
- Permission の引数はセキュアかもしれないけど、ものすごくめんどい。
