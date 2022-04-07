---
title: "Next.jsで環境変数（env）を使いこなすための記事"
emoji: "🧑‍🔧"
type: "tech"
topics: ["nextjs", "nodejs", "typescript", "react", "javascript"]
published: true
---

Next.js を使うプロジェクトでの開発をいくつか経験した結果、環境変数の取り扱いについての知見が溜まりましたので共有します！

:::message
こんな読者の方を対象にしています。

- Next.js 標準の環境変数機能に物足りなさを感じている
- デプロイする環境に合わせて環境変数を切り替えたい。
- 環境変数の TypeScript 型定義が欲しい
- 環境変数に対してテストをしたい
- 複雑な実装やライブラリの導入なしで上記を実現したい

:::

サンプルリポジトリへのリンクを後半に記載しています。実装ベースで確認したい方は後半へどうぞ 💁‍♂️

# Next.js 標準の環境変数機能に不足していること

Next.js には標準で環境変数を管理する機能があります。（以後 `.env` と呼びます。）

https://nextjs.org/docs/basic-features/environment-variables

`.env` の方法は提供されている機能が最小限となっており、次の 3 つのタイミングに合わせてしか環境を切り替えることができません。

| ファイル名         | 読み込まれるタイミング |
| ------------------ | ---------------------- |
| `.env.local`       | 毎回                   |
| `.env.development` | `next dev` 時のみ      |
| `.env.production`  | `next start` 時のみ    |

dev-1, dev-2, staging など複数の環境を切り替えたい場合にこの方法だと少し物足りなくなってきます。
また、定義した環境変数が自動で TypeScript の型定義に反映されることもありません。

以降の章からそんな不足している機能を補う実装を紹介します。

# その前に

標準の `.env` の方法でも複数の環境を切り替えられる方法は一応あります。
ビルド時に反映したい env ファイルを `.env.production` にコピーする方法です。
https://github.com/vercel/next.js/blob/canary/examples/with-docker-multi-env/docker/staging/Dockerfile#L15-L17

これで必要十分な方もいらっしゃるかも知れません。プロジェクト規模や各自の方針に合わせてでどう対応するか検討してみてください。

# デプロイする環境に合わせて環境変数を切り替える

まずこちらを満たす最小構成を作ります。

`env/` ディレクトリを作成し、デフォルトの環境変数ファイルとなる `env/env.local.json` を作成します。

```json:env/env.local.json
{
  "NEXT_PUBLIC_API_BASE_URL": "http://localhost:4000",

  "SECRET_TOKEN": "my_secret_token_for_local"
}
```

以降、環境変数に関係するファイルは `env/` ディレクトリに閉じて配置するようにします。
ディレクトリ名は `env/` 以外でも大丈夫です。

環境の切り替えを確認するために staging 環境のファイルを作成してみます。
JSON だとコメントを挿入できないので不便ですよね？ JavaScript でも作成できます 👍

```js:env/env.staging.js
module.exports = {
  // API サーバーの URL
  NEXT_PUBLIC_API_BASE_URL: "https://staging.example.com",

  SECRET_TOKEN: "my_secret_token_for_staging",
};
```

`next.config.js` にファイルの読み込み設定を追加します。

```diff js:next.config.js
+loadEnv(process.env.APP_ENV);
+
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
};

module.exports = nextConfig;
+
+/**
+ * @param {string} appEnv
+ */
+function loadEnv(appEnv = "local") {
+  const env = {
+    ...require(`./env/env.${appEnv}`),
+    NEXT_PUBLIC_APP_ENV: appEnv,
+  };
+
+  Object.entries(env).forEach(([key, value]) => {
+    process.env[key] = value;
+  });
+}
```

これで最小限の対応は完了しました！
`APP_ENV=staging yarn dev` などと環境を指定して起動することで対象の設定ファイルが読み込まれるようになります。

## 解説

Next.js が起動するとまず `next.config.js` が読み込まれます。そのタイミングで `loadEnv`関数を実行して環境変数ファイルを読み込み、 `process.env` にセットしています。
標準の `.env` の方法とほぼ同じタイミングで環境変数を読み込んでいるため、その後は `.env` でセットされた環境変数と同じように処理されます。

受け取った `APP_ENV` は `NEXT_PUBLIC_APP_ENV` として `process.env` にセットしています。

`NEXT_PUBLIC_` のプレフィックスが付いた環境変数しかブラウザー側に公開されないようになっているため、クライアント（ブラウザー）から参照したい環境変数には `NEXT_PUBLIC_` を付けます。
これは環境変数に設定したシークレットな値が誤ってブラウザー向けのバンドル JS に混入されて流出しないようにするための[Next.js の機能](https://nextjs.org/docs/basic-features/environment-variables#exposing-environment-variables-to-the-browser)です。

参照する環境変数ファイルは APP_ENV 環境変数を渡すことで切り替えられます。

```sh
# ローカル開発
APP_ENV=staging yarn dev

# プロダクションビルド
APP_ENV=staging yarn build
APP_ENV=staging yarn start
```

`next build` `next start` の両方で環境変数を渡す必要があることに注意です。
これはビルド時と起動時の両方で環境変数が必要になるからです。
`NEXT_PUBLIC_` のついた環境変数はビルド時に Webpack.DefinePlugin を用いてバンドル時に置換され、その他の環境変数はランタイムで `process.env` オブジェクトから解決されるためです。[^1]

[^1]: 気になる方は[こちらのコメント](https://zenn.dev/link/comments/f59cdfaa685581)を確認いただくとイメージを掴んでいただけるかも知れません。

:::details Windows の場合
この記事で紹介するコマンドは sh 系のシェルでの動作を想定しているので、Windows では上記のコマンドは動作しないかも知れません。チームメンバーに Windows ユーザーがいる場合は次の対応を検討してみてください。

- WSL や Git bash で開発してもらう
- [cross-env](https://www.npmjs.com/package/cross-env) などを用いてコマンドを共通化する
- Windows のシェルに合った方法で環境変数 `APP_ENV` の代入をおこなってもらう

:::

## 補足

:::message
この方法は @jj さんの[「Next.js における env のベストプラクティス」](https://zenn.dev/jj/articles/next-js-env-best-practice)を参考にして[改良と懸念ポイントなどを調整したもの](https://zenn.dev/link/comments/f59cdfaa685581)です。
:::

# TypeScript 型定義を生成する

環境変数を JSON または JavaScript で定義しているので TypeScript から型としてインポートできるようになります 🎉

`env/types.d.ts` ファイルを作成するだけです。

```ts:env/types.d.ts
type Env = Partial<Readonly<typeof import("./env.local.json")>>;

declare namespace NodeJS {
  interface ProcessEnv extends Env {
    readonly NEXT_PUBLIC_APP_ENV?: string;
  }
}
```

上記のコードでは `env.local.json` の中身を正として扱うようにしています。
また、環境変数がアプリケーション上で実際に定義されているかは保証できないので `Partial` で囲んで undefined の可能性があるようにしています。

# undefined を安全に無視する

上記の型定義で環境変数は Partial (`string | undefined`) として扱われていますが、「この環境変数は絶対定義されているはずだから undefined のハンドリングはしなくて良い」と思う場面はありますよね。

例えば今回の例だと `NEXT_PUBLIC_API_BASE_URL` がないと API との通信ができないのでアプリは何もできません。
そのため変数が undefined であることは気にしなくて良く、特別なハンドリングをする必要もありません。

ただ、変数末尾に `!` (Non-null Assertion Operator) を付けて対応すると、何かの拍子で undefined が入っていた場合でも後続処理に素通りされてしまいます。
予期せぬ不具合に繋がりかねないので、それは避けたいです。

## assertIsDefined を使う

そういった場合には `assertIsDefined` 関数が使えます。
これは TypeScript 3.7 のリリースノートの [Assertion Functions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#assertion-functions) で紹介されていた一つの関数です。

場所はどこでもいいのですが、例として `helpers/assert.ts` を作成してください。

```ts:helpers/assert.ts
export function assertIsDefined<T>(val: T): asserts val is NonNullable<T> {
  if (val === undefined || val === null) {
    throw new Error(
      `Expected 'val' to be defined, but received ${val}`
    );
  }
}
```

この関数に値を渡すと、 `null` または `undefined` の場合はランタイムエラーが発生します。
値がそれ以外の場合は関数は正常に終了し、加えて val に渡した値の型 `T` が次の行以降では `NonNullable<T>` になります。

つまり、undefined のはずがない環境変数を使う時にとりあえず`assertIsDefined` を実行しておけば、 Non-null Assertion Operator と同じように NonNullable 型として扱ってくれるし、万が一 undefined だった場合はエラーにしてくれます。
エラーになりはしますが、後続処理が走ってしまい事故が起きるよりはマシと考えます。

## 使用例

解説が長くなりましたが、以下は React コンポーネントの onClick コールバックで API を叩く例です。

```tsx:/components/someComponent.tsx
import { assertIsDefined } from "helpers/assert";

export const SomeComponent = () => {
  const onClick = () => {
    // string | undefined
    const apiBaseUrl = process.env.NEXT_PUBLIC_API_BASE_URL;

    // 中身が undefined だったらランタイムエラーになる
    assertIsDefined(apiBaseUrl);

    // 値が入っていることが保証された
    // 型は string になっている

    fetch(`${apiBaseUrl}/register`, { method: "POST" });
  };

  <button onClick={onClick}>REGISTER</button>;
};
```

:::message alert
この方法を使うとエラーハンドリングをサボることになりますので、エラーが発生したら対応できるように監視をしておいた方が良いでしょう。
最低限 Sentry や Bugsnag、 NewRelic のようなクラッシュレポート収集サービスを使っておき、エラーの報告があればいつでも修正対応できる状態を作っておくことが大切です。
:::

# テストを書く

環境変数ファイルが多くなってくるとそれぞれでちゃんと定義されているか確認したくなる場合があるでしょう。
Jest を使ってテストを書くことができます。

:::message
[Next.js のドキュメント通りに Jest が設定されている](https://nextjs.org/docs/testing#setting-up-jest-with-the-rust-compiler)ことを前提としています。
:::

```js:env/test.js
import * as fs from "fs";

// Load all defined environment variables
const environmentVariables = fs
  .readdirSync(__dirname)
  .filter((fileName) => fileName.startsWith("env."))
  .reduce((pre, fileName) => {
    pre[fileName] = require(`./${fileName}`);
    return pre;
  }, {});

Object.entries(environmentVariables).forEach(([envName, env]) => {
  describe(`${envName}`, () => {
    test("NEXT_PUBLIC_API_BASE_URL must be set", () => {
      expect(env["NEXT_PUBLIC_API_BASE_URL"]).toBeDefined();
    });
  });
});
```

```text:テスト結果
PASS  env/test.js
 env.dev-1.js
   ✓ NEXT_PUBLIC_API_BASE_URL must be set (1 ms)
 env.dev-2.js
   ✓ NEXT_PUBLIC_API_BASE_URL must be set (1 ms)
env.local.json
   ✓ NEXT_PUBLIC_API_BASE_URL must be set (1 ms)
```

このテストでは全ての環境変数ファイルで `NEXT_PUBLIC_API_BASE_URL` が定義されていることを確認します。

ただこの時点のテストでは Vercel や AWS ECS などの管理コンソールから設定した環境変数のテストはできません。
そう考えるとやはり先ほどの assert の対応の方が利便性が高いかと思います。

# 同じような環境変数ファイルを動的に生成する

dev の確認環境が 1 から 10 まであるとします。`env.dev-[1-10].js` を手動で作成・管理するのはつらいですよね。コピペしたことにより編集が漏れていたりするかも知れません。
もしファイルの内容がほとんど同じなら、異なる部分だけを指定して動的生成してしまえば楽で安全です。

まず、環境変数を生成するためのテンプレート関数を作成します。

```js:env/makeDevEnv.js
module.exports = ({ env, SECRET_TOKEN }) => ({
  NEXT_PUBLIC_API_BASE_URL: `https://dev-${env}-api.example.com`,
  NEXT_PUBLIC_CONTENTS_BASE_URL: `https://dev-${env}-contents.example.com`,

  SECRET_TOKEN,
});
```

次に各環境用のファイルを作成して先ほどの関数に異なる部分のパラメータを渡します。
今回は `dev-1` `dev-2` のみ作ってみます。

```js:env/env.dev-1.js
module.exports = require("./makeDevEnv")({
  env: 1,
  SECRET_TOKEN: "my_secret_token_for_dev-1",
});
// {
//   NEXT_PUBLIC_API_BASE_URL: 'https://dev-1-api.example.com',
//   NEXT_PUBLIC_CONTENTS_BASE_URL: 'https://dev-1-contents.example.com',
//   SECRET_TOKEN: 'my_secret_token_for_dev-1',
// }
```

```js:env/env.dev-2.js
module.exports = require("./makeDevEnv")({
  env: 1,
  SECRET_TOKEN: "my_secret_token_for_dev-2",
});
// {
//   NEXT_PUBLIC_API_BASE_URL: 'https://dev-2-api.example.com',
//   NEXT_PUBLIC_CONTENTS_BASE_URL: 'https://dev-2-contents.example.com',
//   SECRET_TOKEN: 'my_secret_token_for_dev-2',
// }
```

これだけです。JS が使えるとなんでもできてしまいます。
しかし、なんでもできるからといって複雑に継承するようなテンプレート構造とかにしてしまうと剥がしたり分解したくなったときに大変です。
シンプルさは意識しておいた方が良いです。

# サンプルリポジトリ

ここまで紹介した内容のサンプルリポジトリを用意していますので実際に手元で確認していただけます。

https://github.com/aku11i/nextjs-env-management-example

Next.js の初期状態から環境変数機能を追加した差分はこちらです。（紹介した設定をモリモリで実装しています）

https://github.com/aku11i/nextjs-env-management-example/compare/9be2e76...main

# まとめ

Next.js の環境変数周りの tips を紹介しました。
項目別に分離して記載しているので、それぞれのプロジェクト規模や方針に合わせて必要なものを拡張していっていただければと思います。

皆さんの環境変数管理の参考になれば幸いです！
