---
title: "ElectronでのOSコマンドインジェクションの脆弱性事例"
emoji: "🚫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs", "javascript", "electron"]
published: true
---

プライベートで手伝っていた Electron プロジェクトで、OS コマンドインジェクションの脆弱性を発見し、指摘・修正を行いました。

Electron(Node.js) のアプリケーションで、別言語で作成したモジュールを動かす時には `child_process` を使用すると思います。
この記事では `child_process.{exec|execSync}` を使用した際の落とし穴について、Electron での事例を紹介しながら解説します。

# 要約

- exec/execSync はシェルで実行されるため、外部要因に影響するコマンドを実行する場合は必ずサニタイズしなければならない
- クロスプラットフォームで、かつユーザーの動作環境に依存する Electron の場合は OS によって使用されるシェルが変わるため、サニタイズは難しい
- execFile/execFileSync はシェルに依存しないため、可能な限りこちらを使用することが望ましい

# OS コマンドインジェクション

> OS コマンドインジェクションは、ユーザーからデータや数値の入力を受け付けるような Web サイトなどにおいて、プログラムに与えるパラメータに OS への命令文を紛れ込ませて不正に操作する攻撃です。
> https://www.shadan-kun.com/blog/measure/2873/

Web サーバーへの攻撃手法としては一般的に認知されているかと思います。
上記 Web ページで紹介されている例を Node.js で書き換えると下記の通りになります。

```javascript
const { execSync } = require("child_process");

export function sendEmail(address) {
  execSync(`mail -s "タイトル" ${address} < /var/data/aaa.txt`);
}
```

`address` 引数に `xxx@example.com < /etc/passwd #` が渡されると、実行されるコマンドは以下のようになり、`/etc/passwd` の内容がメールアドレスに送信されてしまいます。

```sh
mail -s "タイトル" xxx@example.com < /etc/passwd # < /var/data/aaa.txt
```

これと同じ攻撃を Electron でもできてしまうということを紹介します。

# Electron のセキュリティについて

まず、Electron のセキュリティについての歴史を簡単に説明します。

バージョン 1 や 2 の頃の Electron は、ブラウザー側（レンダラープロセス）に Node.js の API が剥き出しの状態であるため、ひとたび XSS の攻撃を受ければユーザーの PC 全体が危険に晒されてしまう状況でした。

その後バージョンアップと共にセキュリティ対策が行われました。

- Node Integration がデフォルトで無効となる
  レンダラープロセスから Node.js の API にアクセスするのが難しくなりました。
  https://www.electronjs.org/docs/tutorial/security#2-do-not-enable-nodejs-integration-for-remote-content
- preload スクリプトと Context Isolation が導入される
  Node.js の API を使った処理を安全にレンダラープロセスに渡せるようになりました。
  https://www.electronjs.org/docs/tutorial/context-isolation#context-isolation

これらが実装されたことで、仮にアプリに XSS 脆弱性が存在したとしても、OS にまで影響が及ぶ可能性は少なくなりました。

今回は、上記のセキュリティ対策に則って実装しているからといっても、安全なアプリであるとは限らないという話になります。

# Electron での事例

遭遇した事例を元にしたサンプルプロジェクトを作りました。

https://github.com/aktriver/study-electron-os-command-injection

![](/images/2021-08-electron-os-command-injection/preview.png =600x)
_実行画面_

# 説明

入力欄にディレクトリ名を入力し「CREATE」を押すと、`package.json` のあるディレクトリと同階層にディレクトリが作成されるというシンプルなものです。

脆弱性の実装を行なったコミットはこちらです。
https://github.com/aktriver/study-electron-os-command-injection/commit/f4d06fa308485693d07f10c07985180df9e0f75f

# 詳細

Context Isolation に則って実装を行っています。
実装の説明を行なって行きます。

## preload.js

IPC 経由で命令を送信するための関数をレンダラープロセスに登録します。
レンダラープロセスからは `window.electron.createNewDirectory` でアクセスできるようになります。

```javascript
const { ipcRenderer, contextBridge } = require("electron/renderer");

contextBridge.exposeInMainWorld("electron", {
  createNewDirectory: (name) => {
    ipcRenderer.invoke("CREATE_NEW_DIRECTORY", name);
  },
});
```

## レンダラープロセス（renderer.js）

「CREATE」ボタンが押されたら先程の `window.electron.createNewDirectory` を実行します。

```javascript
const createButton = document.getElementById("create");
createButton.addEventListener("click", () => {
  const name = document.getElementById("directory_name").value;
  if (!name) return;

  window.electron.createNewDirectory(name).then(() => {
    alert(`Directory "${name}" was created successfully.`);
  });
});
```

## メインプロセス（main.js）

IPC 経由で命令を受けたらディレクトリを作成します。

```javascript
const { app, BrowserWindow, ipcMain } = require("electron/main");
const { execSync } = require("child_process");

ipcMain.handle("CREATE_NEW_DIRECTORY", (_event, name) => {
  const command = `mkdir "${name}"`;

  console.log("execSync:", command);
  execSync(command);
});
```

:::message
ディレクトリ作成であれば、API で用意されている `fs.mkdir` を使うべきですが、今回はサンプルのため `mkdir` コマンドを実行する形で進めます。
mkdir 以外で実際に起こり得る例では、 `imagemagick` や `ffmpeg` などの外部ツールにファイル名や引数を渡すといったことがあります。
:::

上記の実装では、`mkdir "${name}"` とダブルクオートで囲まれているので、一見安全そうに感じます。

アプリを実行し、ディレクトリ名に `dir1` と入力し、CREATE を押すと、`package.json` と同階層に `dir1` が作成されます。

# 攻撃してみる

ここで攻撃が可能なディレクトリ名を入力してみます。

- Mac の場合
  - `dir2"; open . #`
  - `$(open .; echo dir2)`
- Windows の場合
  - `dir2"; start . || rem "`（未確認です…）

実行すると `dir2` が作成され、同時に何故か Finder/explorer.exe が開くと思います。

`execSync` に渡されたコマンドは変数が展開されて `mkdir "dir2"; open . #"` となっているからです。
（`open .` も `start .` も現在のディレクトリを Finder/explorer.exe で開くコマンドです。）

上記の例では `mkdir "dir2"` と `open .` の 2 つのコマンドが実行されています。

`open .` の部分を任意のコマンドにすることでユーザーの PC を意のままに操作できてしまいます。

今回の実装ではディレクトリ名をユーザーが入力することになっていますが、ディレクトリ名が外部要因で決められる場合は他人に PC を攻撃されてしまうことになります。

# 対策

一番正確で安全な対策は `exec/execSync` から `execFile/execFileSync` に変更することです。

## before

```javascript
const { execSync } = require("child_process");

ipcMain.handle("CREATE_NEW_DIRECTORY", (_event, name) => {
  const command = `mkdir "${name}"`;
  execSync(command);
});
```

## after

```javascript
const { execFileSync } = require("child_process");

ipcMain.handle("CREATE_NEW_DIRECTORY", (_event, name) => {
  execFileSync("mkdir", [name]);
});
```

# exec と execFile の違い

Node.js の公式ドキュメントから今回の説明に必要な部分を引用します。

https://nodejs.org/api/child_process.html

## exec

> The command string passed to the exec function is processed directly by the shell and special characters (vary based on shell) need to be dealt with accordingly:
> （exec 関数に渡されるコマンド文字列はシェルによって直接処理され、特殊文字（シェルによって異なります）はそれに応じて処理する必要があります。）

> Never pass unsanitized user input to this function. Any input containing shell metacharacters may be used to trigger arbitrary command execution.
> （サニタイズされていないユーザーからの入力をこの関数に渡さないでください。シェルメタ文字を含む任意の入力を使用して、任意のコマンド実行をトリガーできます。）

exec は渡された文字列をそのままシェルに処理させるため、パイプやリダイレクトなどシェルの機能が使えます。
macOS/Linux では `/bin/sh -c "コマンド"` で実行しているのと同様であると考えることができます。

そのためシェルで使用可能な `"'#$&|;<()` などの特殊文字はサニタイズする必要がありますが、サニタイズが必要な特殊文字はシェルによって変わるとも記載されています。
Electron では最低でも `/bin/sh` `cmd` `powershell` での実行を考慮する必要があり、サニタイズは現実的ではなさそうです。
動的なコマンドを実行したい場合、 `exec/execSync` はそもそも使用しない方が良さそうです。

## execFile

> The child_process.execFile() function is similar to child_process.exec() except that it does not spawn a shell by default.
> （child_process.execFile（）関数は、デフォルトでシェルを生成しないことを除いて、child_process.exec（）に似ています。）

> Rather, the specified executable file is spawned directly as a new process making it slightly more efficient than child_process.exec().
> （むしろ、指定された実行可能ファイルは新しいプロセスとして直接生成されるため、child_process.exec（）よりもわずかに効率的です。）

> Since a shell is not spawned, behaviors such as I/O redirection and file globbing are not supported.
> （シェルは生成されないため、I / O リダイレクトやファイルグロブなどの動作はサポートされていません。）

`execFile` では一つ目の引数に渡した実行ファイルから直接プロセスが生成されるため、シェルの構文が挿入される危険性がありません。

また、シェルで実行されないため、環境変数も展開されません。
`execFile('mkdir', ["$HOME"])` と実行しても、`$HOME`は展開されずに文字列のまま渡されます。

シンプルな 1 コマンドで完結し、引数がユーザーや外部の要因により動的に変動する場合には `execFile` を使用するのが無難です。
また、第一引数のコマンドも、 `/bin/mkdir` と絶対パスで指定した方がより安全です。

# 最後に

exec/execFile の違いについては Node.js の話なので、Electron に限るわけではありません。
インジェクション系の脆弱性は気を抜いていると入り込んでしまうことがあるなと思いました。
注意したいところです。
