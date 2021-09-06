---
title: "ボタンに吸い付くポインターを実装してみる（JavaScript）"
emoji: "😚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "html", "css"]
published: false
---

もう 1 年半ほど前になりますが、 iPad OS 13.4 からマウスなどのポインティングデバイスがサポートされ、接続した時にポインターが表示されるようになりました。

このポインターは要素に合わせて形状を変えるようになっており、ボタンに吸い付いたりテキストカーソルの形に変形したりします([参考動画(Youtube)](https://youtu.be/9iO52-MIBP0?t=51))。

触っていてとても気持ち良いため、似たようなものを Web ページ向けに JavaScript で実装してみました。

https://twitter.com/aku11i/status/1430803913876271109

この記事では上記ツイートの動画と同じ動作をするポインターの実装手順を紹介します。

:::message
ライブラリとしても公開しています。
簡単に Web ページに組み込んでみたい方はこちらをご覧ください！

https://github.com/aku11i/kimochii-pointer
:::

# サンプルプロジェクト

今回紹介する実装の完成形を Codepen で公開しています。

https://codepen.io/aku11i/pen/powEJmP

# 実装

ポインターを実装します。
要素の作成・スタイルの適用まで JavaScript で行います。

# マウスポインターを作る

まず、シンプルなマウスポインターを作成します。
ポインターとなる div 要素を作成します。

```javascript
const pointer = document.createElement("div");

// 見た目を整える
pointer.style.width = "20px";
pointer.style.height = "20px";
pointer.style.borderRadius = "50%";
pointer.style.backgroundColor = "gray";
pointer.style.opacity = "0.5";

// 他の要素の上に表示されるように `z-index` の値を上げる
pointer.style.zIndex = "100000";

// 初期位置をページの真ん中にする
pointer.style.position = "absolute";
pointer.style.transform = "translate(-50%, -50%)";
pointer.style.top = "50%";
pointer.style.left = "50%";

// ポインターに対してのマウスイベントを透過させる
pointer.style.pointerEvents = "none";

// ページに表示する
document.body.append(pointer);
```

ページの真ん中に丸いグレーのポインターが表示されるようになりました。

# マウスに合わせて移動させる

`mousemove`のイベントを受け取ったらポインター要素を同じ位置に移動するようにします。

```javascript
window.addEventListener("mousemove", (event) => {
  const { pageX, pageY } = event;

  // マウス座標に移動させる
  pointer.style.top = pageY + "px";
  pointer.style.left = pageX + "px";
});
```

これでひとまずマウスの移動に合わせて動くシンプルなポインターが完成します。
まだデフォルトのマウスカーソルが見えている状態ですが、隠し方については終盤で紹介します。

# 要素に貼り付くようにする

ポインターが特定の要素の上に移動した際に貼り付くようにします。
今回は `sticky` というクラス名が付いている要素に貼り付くようにしてみます。

先ほどの `mousemove` イベントの処理を書き換えます。

```javascript
window.addEventListener("mousemove", (event) => {
  const { pageX, pageY, clientX, clientY } = event;

  // マウス下の要素一覧を取得
  const elements = document.elementsFromPoint(clientX, clientY);
  // `sticky` クラスが付いている要素を探す
  const target = elements.find((el) => el.classList.contains("sticky"));

  if (target) {
    // sticky要素があった時はポインターを要素と同じ場所・大きさに変形させる
    const rect = target.getBoundingClientRect();
    const top = rect.top + rect.height / 2;
    const left = rect.left + rect.width / 2;
    const { width, height } = rect;
    const borderRadius = Math.min(rect.height, rect.width) * 0.1;

    pointer.style.top = top + "px";
    pointer.style.left = left + "px";
    pointer.style.width = width + "px";
    pointer.style.height = height + "px";
    pointer.style.borderRadius = borderRadius + "px";
  } else {
    // sticky要素がない場合は元の形状に戻す
    pointer.style.width = "20px";
    pointer.style.height = "20px";
    pointer.style.borderRadius = "50%";

    // マウス座標に移動させる
    pointer.style.top = pageY + "px";
    pointer.style.left = pageX + "px";
  }
});
```

動作を確認するためのボタン要素を用意します。

```html
<div>
  <button class="sticky">♠️</button>
  <button class="sticky">♣️</button>
  <button class="sticky">♦️️</button>
  <button class="sticky">️♥️</button>
</div>

<style>
  button {
    font-size: 40px;
    width: 60px;
    height: 60px;
    border-width: 0px;
    background-color: transparent;
    margin: 5px;
  }
</style>
```

ボタンの上にポインターを移動させると対象の要素と同じ大きさに変形し、ぴったりと固定されるようになりました。

ここまでの実装を解説していきます。

## マウス下の要素の取得について

マウスと同じ座標にある要素を取得するために [`document.elementsFromPoint`](https://developer.mozilla.org/ja/docs/Web/API/Document/elementsFromPoint) を使用しています。

```javascript
const elements = document.elementsFromPoint(clientX, clientY);
```

指定した座標に重なる要素を配列形式で返してくれる API で、引数の座標には `clientX/Y` を使用します。これはページ全体の座標(`pageX/Y`)からスクロールの差分(`scrollX/Y`)を引いたものです。

実行結果について、例えば `<div><a><img></a></div>` という構造のドキュメントで `img` 要素がある座標を指定した場合、 `[img, a, div, body, html]` という実行結果を得ることができます。

この配列の中から `sticky` のクラス名を持つ要素を探します。

```javascript
const target = elements.find((el) => el.classList.contains("sticky"));
```

`target` が `undefined` でなければ `sticky` のクラス名を持つ要素の上にマウスがあるということになります。

## 要素に貼り付かせる

ポインターを対象要素に貼り付くようにします。
対象の要素と同じ大きさにポインターを変形させることで貼り付いたように見せることができます。

```javascript
const rect = target.getBoundingClientRect();
const top = rect.top + rect.height / 2;
const left = rect.left + rect.width / 2;
const { width, height } = rect;
const borderRadius = Math.min(rect.height, rect.width) * 0.1;

pointer.style.top = top + "px";
pointer.style.left = left + "px";
pointer.style.width = width + "px";
pointer.style.height = height + "px";
pointer.style.borderRadius = borderRadius + "px";
```

[`element.getBoundingClientRect`](https://developer.mozilla.org/ja/docs/Web/API/Element/getBoundingClientRect) でページ全体から見た要素の位置を取得することができます。

角を少し丸くしたいので、`borderRadius` は対象要素の短い方の辺の 10% となるよう指定しています。

```javascript
const borderRadius = Math.min(rect.height, rect.width) * 0.1;
```

## ポインターを元に戻す

対象の要素が見つからない場合はポインターを元の形状に戻します。
貼り付かせる時に変更したスタイルを初期の値に戻します。

```javascript
pointer.style.width = "20px";
pointer.style.height = "20px";
pointer.style.borderRadius = "50%";
```

座標もマウスの位置と同じになるように更新します。

```javascript
pointer.style.top = pageY + "px";
pointer.style.left = pageX + "px";
```

# アニメーションを付ける

ここまでで要素に貼り付くポインターが完成しましたが、まだアニメーションがありません。
「貼り付く」から「吸い付く」ような表現になるようにアニメーションを導入していきます。

今回、アニメーションは CSS ではなく JavaScript で実装していきます。
理由は CSS で試してみたところ、ブラウザーによって期待通りの動作にならないことがあったためです。

例えば [この例(検索でヒットしたものを拝借)](https://codepen.io/ddryo-the-encoder/pen/BaBYZdW) は CSS アニメーションで実装されており、Chrome では問題なく動作していますが、 Safari で確認するとポインターの移動がスムーズではありません。

# GSAP をインストールする

JavaScript 向けのアニメーションライブラリである[GSAP (GreenSock Animation Platform)](https://github.com/greensock/GSAP) を使用します。
Web でリッチなアニメーション表現を行う際によく利用されるライブラリのようです。
私は今回初めて使用しました。

```sh
npm install --save gsap@3.x
# yarn add gsap@3.x
```

[`gsap.to()`](<https://greensock.com/docs/v3/GSAP/gsap.to()>)を使用することでスタイルの適用にアニメーションを付けることができます。

# アニメーションを組み込む

GSAP を読み込みます。

```javascript
import gsap from "gsap";
```

`mousemove` イベントの中のスタイルを適用している箇所を `gsap.to` に置き換えます。

```diff javascript
  if (target) {
    // sticky要素があった時はポインターを要素と同じ場所・大きさに変形させる
    const rect = target.getBoundingClientRect();
    const top = rect.top + rect.height / 2;
    const left = rect.left + rect.width / 2;
    const { width, height } = rect;
    const borderRadius = Math.min(rect.height, rect.width) * 0.1;

-    pointer.style.top = top + "px";
-    pointer.style.left = left + "px";
-    pointer.style.width = width + "px";
-    pointer.style.height = height + "px";
-    pointer.style.borderRadius = borderRadius + "px";
+    gsap.to(pointer, {
+      top,
+      left,
+      width,
+      height,
+      borderRadius,
+      duration: 0.1,
+      overwrite: true,
+    });
  } else {
    // sticky要素がない場合は元の形状に戻す
-    pointer.style.width = "20px";
-    pointer.style.height = "20px";
-    pointer.style.borderRadius = "50%";
+    gsap.to(pointer, {
+      width: 20,
+      height: 20,
+      borderRadius: "50%",
+      duration: 0.1,
+      overwrite: true,
+    });

    // マウス座標に移動させる
-    pointer.style.top = pageY + "px";
-    pointer.style.left = pageX + "px";
+    gsap.to(pointer, {
+      top: pageY,
+      left: pageX,
+      duration: 0.05,
+    });
  }
```

`gsap.to` には値を number 型で渡せるので、 `"px"` を結合する必要がなくなりました。

`duration` はアニメーションが持続する秒数です。
`gsap.to(element, { width: 100, duration: 3 })` だと 3 秒かけて要素の横幅を `100px` にするという意味になります。

`overwrite` は他に適用中のアニメーションがあればそれをキャンセルします。

これで実装としては完成で、冒頭に貼ったツイートの動画と同じような動きになりました。

# デフォルトのマウスカーソルを隠す

（お好みで）OS デフォルトのマウスカーソルを非表示にします。
全ての要素に対してマウスカーソルを非表示にするスタイルを CSS に記述します。

```css
* {
  cursor: none !important;
}
```

# ポインターの移動遅延について

ポインターを操作していると、実際のマウスカーソルと比べて移動に遅延が生じていることに気付くかもしれません。
GSAP でポインターの移動に対してもアニメーションを適用しているためです。

```javascript
gsap.to(pointer, {
  top: pageY,
  left: pageX,
  duration: 0.05,
});
```

理由はポインターの座標が急に飛んでしまうことがあるためです。
ポインターが要素に吸い付いている時は、座標は常に対象要素の中心にあります。

```javascript
// sticky要素があった時はポインターを要素と同じ場所・大きさに変形させる
const rect = target.getBoundingClientRect();
const top = rect.top + rect.height / 2;
const left = rect.left + rect.width / 2;
```

ポインターが要素から外れる時に `top` と `left` の位置が急に要素の外に移動してしまうので、この座標移動にアニメーションが入るようにしないと外れる時の見え方に違和感が生まれてしまいます。

そのため、操作してて気にならない程度にポインターに移動遅延を適用しています。

## マウスストーカー

先程のマウスの移動処理の `duration` を `0.2` などに増やしてみると、ポインターがマウスの移動にゆっくりついてくるようになったと思います。
これをマウスストーカーを呼びます。
つまりこのポインターの正体は追従速度を速くしたマウスストーカーということになります。

変形処理の `duration` も `0.2` にしてみるとマウスストーカーとしても面白い動作をすると思います。

# 類似実装

同じようにボタンに吸い付くポインターの実装事例が見つかりましたので紹介します。

https://pavellaptev.github.io/context-cursor/

アニメーションに GSAP を使用しているなど、基本的な作り方は同じですが、表現がより iPad のポインターに近く、触っていて楽しいです。参考になりました。

- [GitHub Repo](https://github.com/PavelLaptev/context-cursor)
- [解説記事](https://blog.prototypr.io/ipad-pointer-on-the-web-f3aaf48d515c)

# ライブラリの紹介

今回紹介した実装も含めたライブラリを [npm package](https://www.npmjs.com/package/kimochii-pointer) として公開しています。
簡単に始めてみたい方はこちらを使ってみてください！

https://github.com/aku11i/kimochii-pointer

[オリジナルの変形処理を追加する](https://github.com/aku11i/kimochii-pointer#create-a-custom-shape) こともできます。

# 最後に

アニメーションのパラメータを調整したり要素の動かし方を変えてみたり、少し工夫するだけでもっと良いものが出来上がると思います。

皆さんも工夫して気持ちいいポインターを作成してみてください！
