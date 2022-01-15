---
title: "Reactコンポーネントを手続き的に呼び出す"
emoji: "✍️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "javascript", "typescript"]
published: true
---

React で手続き的にコンポーネントを呼び出す仕組みを作ってみました。

# 背景

きっかけはダイアログの表示処理をもっとシンプルにできないかと考えたことです。

例えばユーザーが削除ボタンを押した際に本当に削除するかの確認ダイアログを表示するとします。
ブラウザー備え付けの `window.confirm` を使うとこのようになります。

```tsx
const handleClickDelete = () => {
  const result = window.confirm("本当に削除しますか？");
  if (result) deleteItem(item.id);
};

<DeleteButton onClick={handleClickDelete} />;
```

同様に「はい」「いいえ」を確認するようなダイアログを React コンポーネントで実装すると、呼び出し側の実装は下記のようになります。

```tsx
const [opened, setOpened] = useState(false);

const handleClickDelete = () => {
  setOpened(true);
};

const handleClickYes = () => {
  deleteItem(item.id);
  setOpened(false);
};

const handleClickNo = () => {
  setOpened(false);
};

<DeleteButton onClick={handleClickDelete} />
<ConfirmDialog
  open={opened}
  message="本当に削除しますか？"
  onClickYes={handleClickYes}
  onClickNo={handleClickNo}
/>
```

`window.confirm` と比較してステート管理とコールバック関数が増えました。
また、この例だと `deleteItem` に辿り着くまでに `handleClickDelete` → `setOpened/opened` → `ConfirmDialog` → `handleClickYes` を順に見ていく必要があります。

このような確認ダイアログのコンポーネントを `window.confirm` と同じように手続き的に呼び出せたら便利ではないでしょうか？
下記のような具合です。

```tsx
const handleClickDelete = () => {
  const message = "本当に削除しますか？";
  const result: boolean = renderComponent(ConfirmDialog, { message });
  if (result) deleteItem(item.id);
};

<DeleteButton onClick={handleClickDelete} />;
```

上記のようにコンポーネントを手続き的にレンダリングし、Promise で結果を受け取れるようにする hook を作ってみました。

# ライブラリ

ライブラリ化まで行ったものをこちらで公開しています。

https://github.com/aku11i/react-procedure

以下で解説など行っていきます。

# 解説

:::message
ここで記載するコードは一部のみ記載しているので、そのままコピペするだけでは動作しません。
予めご了承ください。
:::

今回手続き的に呼び出したい `ConfirmDialog` のインターフェースはこちらです。

```tsx
export type ConfirmDialogProps = {
  message: string
  onClickYes: () => void
  onClickNo: () => void
}

export const ConfirmDialog: VoidFunctionComopnent<ConfirmDialogProps> = (
  // 中身は重要でないので省略します
```

この `ConfirmDialog` を先ほど載せた例のように表示したいです。
まずは `renderComponent` 関数を作成します。

```tsx
const [element, setElement] = useState<JSX.Element>();

const renderComponent = (
  component: typeof ConfirmDialog
  props: DialogConfirmProps
): JSX.Element => {
  const el = React.createComponent(component, props);
  setElement(el);
};

const handleClickDelete = () => {
  const element = renderComponent(ConfirmDialog, {
    message: "本当に削除しますか？",
    onClickYes: () => {
      deleteItem(item.id);
      setElement(undefined)
    },
    onClickNo: () => {
      setElement(undefined)
    },
  });
};

<DeleteButton onClick={handleClickDelete} />
<div>{element}</div>
```

`renderComponent` はコンポーネントから要素を生成し、`setElement` を介して要素をレンダリングします。
`DeleteButton` をクリックすれば `<div>{element}</div>` に `ConfirmDialog` がレンダリングされます。

`onClickYes/No` をコールバック関数で渡しているところが中途半端に感じます。
yes/no の結果は `renderComponent` の戻り値で受け取りたいです。
`renderComponent` を書き換えてみます。

```tsx
const renderComponent = (
  component: typeof ConfirmDialog,
  props: Omit<DialogConfirmProps, "onClickYes" | "onClickNo">
): Promise<boolean> => {
  return new Promise<boolean>((resolve) => {
    const onClickYes = () => {
      setElement(undefined);
      resolve(true);
    };
    const onClickNo = () => {
      setElement(undefined);
      resolve(false);
    };

    const el = React.createComponent(component, {
      ...props,
      onClickYes,
      onClickNo,
    });

    setElement(el);
  });
};
```

Promise を使用してコールバック関数のどちらかが実行されるのを待ってみました。
どちらかが呼ばれると Promise が解決されます。

こうすることで呼び出し側はとてもシンプルになります。

```tsx
const result = await renderComponent(ConfirmDialog, {
  message: "本当に削除しますか？",
});
```

また、`renderComponent` に関連する処理は hooks に切り出すことができます。

```tsx
const useConfirmDialog = () => {
  const [element, setElement] = useState<JSX.Element>();

  const renderComponent = (
    props: Omit<DialogConfirmProps, "onClickYes" | "onClickNo">
  ): Promise<boolean> => {
    return new Promise<boolean>((resolve) => {
      const onClickYes = () => {
        setElement(undefined);
        resolve(true);
      };
      const onClickNo = () => {
        setElement(undefined);
        resolve(false);
      };

      const el = React.createComponent(ConfirmDialog, {
        ...props,
        onClickYes,
        onClickNo,
      });

      setElement(el);
    });
  };

  return { renderComponent, element };
};
```

最終的に呼び出し側のコードはこのようになり、`ConfirmDialog` を利用しやすくなりました。

```tsx
const { renderComponent, element } = useConfirmDialog();

const handleClickDelete = async () => {
  const message = "本当に削除しますか？";
  const result: boolean = await renderComponent({ message });
  if (result) deleteItem(item.id);
};

<DeleteButton onClick={handleClickDelete} />;
<div>{element}</div>;
```

# 注意点

今回の実装では `handleClickDelete` 関数の中で `renderComponent`(`React.createElement`) を実行しています。
ここで `ConfirmDialog` に渡される props はリアクティブ性が失われているので、例えば下記のように `message` をステートにしてどこかで更新されることを期待しても反映されません。
`message` は `renderComponent`（正確には`handleClickDelete`）が呼び出される時点の値で確定されてしまいます。

```typescript
const [message, setMessage] = useState("");

const handleClickDelete = async () => {
  const result: boolean = await renderComponent({ message });
  if (result) deleteItem(item.id);
};
```

ということもあって潜在的な不具合を生み出す可能性があるので、仕組みは作ってみたものの利用するかどうかは迷っている状況です。

# 使い所

個人的に思う使い所としては、レンダリングの最中に props の変更を気にしなくて良いような

- ダイアログ全般
- アニメーションエフェクト
  - ボタンを押した時のパーティクルエフェクト
  - Twitter の誕生日おめでとうみたいなやつ

などに使えそうかなと感じています。

また、ゲームのステージ管理で router ライブラリの代わりとかにも使えそうかもと感じています。
↓ こんな感じです。

```tsx
const stage1 = () => {
  const { cleared } = await renderComponent(Stage1);
  cleared ? stage2() : topPage();
};
```

# 最後に

今回の解説実装では `ConfirmDialog` という特定コンポーネントに依存した hook になってしまっていますが、初めに紹介したライブラリでは共通インターフェースを用意することで汎用的に利用できるようになっています。
興味がありましたら見てみてください！

https://github.com/aku11i/react-procedure
