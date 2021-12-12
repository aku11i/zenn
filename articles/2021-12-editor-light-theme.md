---
title: "ライトテーマのすすめ OS設定に合わせてカラースキームを切り替える"
emoji: "🌞"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "vim", "neovim", "intellij", "terminal"]
published: true
---

:::message
この記事は [Uzabase Advent Calendar 2021](https://qiita.com/advent-calendar/2021/uzabase) 12 日目の記事です。
:::

仕事環境に応じてカラースキームを変えてみて良かったという話をします。
「エディターのカラースキームは昔から変えていない」「ダークテーマ一択！」という方に是非見てもらいたい記事です！

![editor](/images/2021-12-editor-light-theme/switch-vscode.gif)

# 概要

macOS の外観モード（ライト / ダーク）の設定に応じて各種開発ツールのカラースキームを変更する方法をまとめました。
VSCode / IntelliJ に関しては Windows / Linux(GNOME) でも使えるかも知れません。

## ページ内リンク

- [iTerm2](https://zenn.dev/aktriver/articles/2021-12-editor-light-theme#iterm2)
- [VSCode(Visual Studio Code)](<https://zenn.dev/aktriver/articles/2021-12-editor-light-theme#vscode(visual-studio-code)>)
- [IntelliJ IDEA](https://zenn.dev/aktriver/articles/2021-12-editor-light-theme#intellij-idea)
- [Vim/Neovim](https://zenn.dev/aktriver/articles/2021-12-editor-light-theme#vim%2Fneovim)

# 経緯

私事ですが、最近 PC の画面を集中して見続けることができず少し悩まされていました。
画面を見ているとすぐに疲れてきて、目薬をさしたり気分転換したりを頻繁に行っていました。

この現象が起こるのは昼間だけで、夜になるにつれて開発に集中できるようになります。
私は夜行性タイプではありますが、前職では 9 時起きの 10 時出社で問題なく開発スタートを切れていたため、体質については一旦考えないことにしました。
そこで現職の株式会社アルファドライブに転職してからのフルリモート勤務による物理的な仕事環境の変化を疑いました。

今の仕事部屋は東と南から日光を取り入れることができるため、日当たりが良いです。
日差しに負けないように部屋の照明の明るさも最大にしています。

ここでディスプレイの明るさが部屋の明るさに対して負けていることに気づきました。[^1]
おまけに私はダークモードを強く好み、白い背景は目に毒とばかりに普段使うアプリケーションは全て黒背景で統一していました。[^2]
ディスプレイの明るさが弱い状態でダークモードを使用していることで画面が暗くなり、返って目を酷使してしまっている状況であると考えました。

:::message alert
個人の考えです。医学的根拠はありません。
:::

そこでダークモードへの固執をやめ、ライトモードを使って明るい画面にしてみることにしました。
まだ実践してみて数日ほどで環境変化による一時的な効果の可能性も捨て切れませんが、今のところ良い影響があります。
ダークモードと比べて昼でも集中できるようになりました。

とはいえ夜になり部屋の明るさが下がってきた時にライトモードの明るい背景は目に刺激的です。
macOS には日の出・日の入りに応じて OS のダークモード・ライトモードの設定を自動的に変更する機能がありますので、これに応じてエディタや開発ツールのカラースキームも変更されるようにしました。

OS に合わせてダークモード・ライトモードを切り替える設定はデフォルトでは有効になっていないことが多いです。
自分の使用している主要な開発ツールでの設定方法を調べたので他の方にも設定してみて欲しくまとめました。

[^1]: 使用しているモニターディスプレイの最大輝度は 350 ニトでした
[^2]: Web ページも Chrome 拡張機能の Dark Reader を使ってダークモードにしているほどです。

# 使用しているカラースキーム：Solarized

![solarized](/images/2021-12-editor-light-theme/solarized.png)

私は以下の理由から [Solarized](https://ethanschoonover.com/solarized/) をメインのカラースキームとして使用しています。

- 配色が目に優しい
- 有名なカラースキームであり、どのエディターでもプリインストールされているか、拡張機能として公開されている（`[エディタ名] solarized` でググってみてください）
  → ターミナルと Vim などで背景色を揃えることができる

明るい背景の Solarized Light と暗い背景の Solarized Dark が用意されています。
初めは Solarized Dark の青い背景が少しキツく感じるかも知れませんが、すぐに慣れると思います。
Light の方は今まで興味ありませんでしたが、こちらも目に優しい配色となっているため今回使用しています。

# OS の外観設定に合わせてカラースキームを切り替える

いよいよ macOS の設定に合わせてカラースキームを切り替える方法を記載します。
だいたいは公式の設定で対応できます。

## iTerm2

![iterm2](/images/2021-12-editor-light-theme/switch-iterm2.gif)

本記事記載時点の `iTerm2 3.4.14 (OS 10.14+)` では機能として存在しませんでした。
Beta 版の `iTerm2 3.5.0beta3 (OS 10.14+)` では設定に "Use different colors for light mode and dark mode" という項目が追加されており、こちらで切り替えることができます。

![iterm2 settings](/images/2021-12-editor-light-theme/settings-iterm2.png)

iTerm2 のベータ版はこちらからダウンロードできます。
https://iterm2.com/downloads.html

## VSCode(Visual Studio Code)

![vscode](/images/2021-12-editor-light-theme/switch-vscode.gif)

設定ファイル `~/Library/Application Support/Code/User/settings.json` に以下のように記述します。

```json
{
  "window.autoDetectColorScheme": true,
  "workbench.preferredDarkColorTheme": "Solarized Custom Dark Italic",
  "workbench.preferredLightColorTheme": "Solarized Custom Light Italic"
}
```

テーマ名はお好みのものに変更してください。
私が使っているものはこちら
https://marketplace.visualstudio.com/items?itemName=bbrakenhoff.solarized-light-custom

## IntelliJ IDEA

![intellij idea](/images/2021-12-editor-light-theme/switch-intellij.gif)

設定画面に `Sync with OS` という項目がありました。

![intellij idea settings](/images/2021-12-editor-light-theme/settings-intellij.png)

## Vim/NeoVim

![neovim](/images/2021-12-editor-light-theme/switch-neovim.gif)

こちらのみ拡張機能での対応になります。
[vim-auto-color-switcher](https://github.com/kat0h/vim-auto-color-switcher) というプラグインがありましたのでこちらを使わせていただきました。
ダークモードの切り替えに応じて `set background=dark` `set background=light` を実行してくれます。
オプションでカラースキームの切り替えコマンドを実行させることもできるようです。

↓ 紹介記事
https://zenn.dev/kato_k/articles/3f1abb1f83419e

ただ、内部で使われている API が NeoVim と互換性がないようで少し調整を行う必要がありました。

```diff:vim-auto-color-switcher/plugin/auto_color_switcher.vim
-  if a:msg == "light"
+  if a:msg == "Light"

-let s:job = job_start(s:exe, {"out_cb": function('s:CallBack')})
+let s:job = jobstart(s:exe, {"on_stdout": {j,d -> s:CallBack(j, d[0])}})
```

こちらに関してまた PR を出したいと考えています。

また、 vim での Solarized プラグインは [vim-solarized8](https://github.com/lifepillar/vim-solarized8) がオススメです。
`set background` が `light` か `dark` かでテーマの切り替えを行ってくれるので、先程のプラグインとの相性も良いです。

# 最後に

各種開発ツールで OS の設定に合わせてカラースキームの切り替えを行う方法を紹介しました。
エディターのカラースキームは今回のように眼精疲労に影響したりすることもありますし、好みのカラースキームだとコーディングの調子が上がって生産性の向上にも幾分か効果が見られると思います。
年末休暇も近づいていますし、皆さんもこの機会にパフォーマンスの出せるテーマ・カラースキームの設定を考えてみてください！
