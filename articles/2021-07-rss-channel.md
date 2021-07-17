---
title: "社員の技術インプットを向上させるためにRSSチャンネルを導入して1年運用した"
emoji: "🗞️"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["rss", "javascript", "nodejs"]
published: false
---

技術系のブログやニュースサイトの新着記事を流すチャンネルを社内チャットに作りました。
1 年運用してみたので、導入の経緯や結果として社内にどういった変化があったかをまとめてみます。

他社の事情を知らないのですが、あまりやってない取り組みなんじゃないかと思っています。
参考になれば幸いです。

# どんなもの？

![](https://storage.googleapis.com/zenn-user-upload/2136cbb4c552d3b4ce84e373.png)

こんな感じで、登録しているサイトの新着記事をチャットに投稿して知らせてくれます。

# どんな情報が流れる？

現職では Chatwork が標準のチャットツールとなっており、そこで合計 5 つの RSS チャンネル[^1]が稼働しています。

![](https://storage.googleapis.com/zenn-user-upload/3fa7146690459f2750e2a14d.png =200x)

なるべく社内の部署毎にニーズを満たせるようにチャンネルを分けていますが、結局参加してくれている人はほぼ全部に入ってくれています。

[^1]: Chatwork では「チャンネル」でなく「ルーム・グループ」と言いますが、Slack や Teams に合わせて「チャンネル」と呼びます。ごめんなさい。

### 技術ニュース

Publickey などの技術系ニュースサイトや Apple・Google・Docker・TypeScript・React などのオフィシャルブログを登録しています。

また、はてブの総合トップから Qiita や Zenn、SpeakerDeck など技術サイトのドメインを絞り込んで投稿していたりもします。

### 技術ブログ

個人・企業問わず参考になる技術ブログやポッドキャストを登録しています。

### デザイン知識

CSS や UI/UX・CG など、デザイナーに有益な情報

### IT 知識

スマホ・PC・OS などの新着情報
スマホゲーの開発もしているので、変態スマホとかにはアンテナを張っておかないといけないです。

### 業界情報

業界ニュースや取引先・競合他社のプレスリリースなど

# 導入理由

社内の開発メンバーのインプットを向上させたかったのが大きな理由です。
面談の時に上司が「ウチには職業プログラマー[^2]が多い」みたいなことを漏らしていたので、それが少しでも改善されれば良いと思っていました。

また、自身へのメリットとしては、社内に浸透していない技術の採用を提案する時に「何それ？聞いたことない」より「ああなんとなく知ってる！」と反応が返ってくる方が話をスムーズに進められると思ったのもあります。

学生時代に Slack を使って仲間内で同じことをやっていたので、上記の解決手段として RSS チャンネルを思いついたわけです。

[^2]: 仕事でしかプログラミングをせず学習意欲がないプログラマーのことを指していると思います。

# 提案から運用までの流れ

## 提案

昨年 4 月に次のような内容をチャットに投稿しました。ちょうど緊急事態宣言下でテレワーク中でした。

> 技術ブログの更新や使用しているサービスのお知らせを通知する RSS チャンネルがあったらいいと思いませんか？
> 技術記事などの新着情報が Chatwork に流れることで、興味がそれほどない人でも技術動向に関心を持ったり知見を広めたりすることが期待できると思います。

この投稿に対し、偉い人から「👍」のリアクションがついたのでとりあえず動かしてみることにしました。

ちなみに RSS を導入する以前にも気になった記事を見つけたら社内チャットに共有する文化はありましたが、投稿しても誰も反応しないこともあったりしてあまり盛んではありませんでした。

## 開発

Slack だったら[公式 App で一瞬](https://slack.com/apps/A0F81R7U7-rss)ですが、Chatwork ではそのような機能はありませんでした。

初めは IFTTT とかを使ってノーコードで簡単に動かそうと思いましたが、誰かが記事ソースの追加・削除をしたい時にアカウントの権限周りが障壁となる気がしてやめました。
ちょうど案件の納品を終えて一息付いたタイミングだったので、自分で Node.js でコーディングして社内の Gitlab で管理することにしました。

運用を続けていると後から投稿時間や単語のフィルター（「転職」「募集」など）などカスタマイズが必要になってくると思うので、時間があればオリジナルで実装するのが良いと思います。

作り始めたのが金曜日だったので、とりあえず動く状態まで実装して土日を使って動作テストしました。
投稿するチャンネル名と RSS の記事ソースを JSON に定義して、定期的にフェッチします。
新着記事があれば Chatwork に投稿するというシンプルなものです。

最初の記事ソースは私が個人的に購読していたものから選んで、あとは会社の技術スタックに合いそうな物をいくつか探して登録しました。

後から誰でも記事ソースを追加できるように、json に URL を追記してコミットすれば CI/CD で自動反映してくれるようにしました。

## 公開

月曜日になってから、プログラマーだけが参加しているチャンネルでテスト運用中の旨をさりげなく投稿しました。
5-10 名ほどが参加してくれました ✌️

改良を加えつつ 2 ヶ月ほど運用して安定稼働することを確認できたので、公開範囲をデザイナーにも広げました。
普段どんなサイトでデザイン関係の情報を収集しているかをヒアリングして、教えてもらったサイトを追加してから連絡しました。

そこから更に 1 ヶ月稼働させて、許可が降りたため全社チャットで公開する運びとなりました。
今では社員数 40 名ほどに対し、20-30 名が各チャンネルに参加してくれています 🎉

# 1 年間運用してみて良かった点

運用してみて社内に起きた変化をまとめます。

## 重要な情報を見逃さなくなった

プラットフォームの仕様変更の通知などを見逃さなくなりました。

- Docker Hub が無料ユーザーの pull 回数の制限を始める
- 2021 年 8 月以降新規で公開する Android アプリは AAB(Andoird App Bundle)の形式で公開しなければならない

などの重要な情報が流れた時に、記事を読んだ人が他のチャットに転載して注意喚起したり、会議の議題に上げてくれたりします。
発表時点で知ることができるため、対策するための期間も確保することができます。

## 引用されるようになった

社内の会話や議論の中で記事が引用されるようになりました。

「○○ イマイチ分からない」「最近 RSS に流れてたこの記事良かったよ」など

## 社員の技術情報への興味・関心が上がった

効果測定を行えていないので感覚の話になってしまいます。
実際に RSS 経由で知ったかの確認ができていませんが、

- Web デザイナー（コーダー）から Tailwind CSS の共同検証を提案された。
- Android や iOS の最新 API の情報が社内に周知されやすくなった。

といった効果がありました。

# 残念だった点

## もっと反応が欲しかった

投稿された記事に対して引用やリプライなどでその記事に対する見解を投稿するなど、社員同士での議論が行われることを期待していました。

私も自分の気になった記事を引用してコメントを付けたり、リアクション（👍）を付けたりしてみましたが、後に続いてくれる方はあまり現れませんでした。

とりあえずインプット向上の目的は満たせましたが、アウトプットまで促せるような施策があればなお良かったかも知れません。

# 懸念点

まだ立証できていない懸念もあります。

## 業務を妨げていないか

導入前と導入後の比較をしていないので、社員が記事に夢中で業務に集中できなくなっている可能性は否定できません。

四六時中投稿されていては社員の集中力を妨げる可能性があるため、上司と相談して今は日中の毎時 20 分と 50 分に投稿するようにしています。
もし悪影響があれば業務時間の前後や休憩時間に合わせて投稿するなど、投稿時間の工夫で解消できると思います。

## 社員の転職意欲を掻き立てないか

記事のタイトルや冒頭に「採用」「転職」「募集」などが含まれている場合は投稿しないようにしていますが、企業の技術ブログは最後に採用案内が載っていることがほとんどです。
それが社員に影響を及ぼす可能性は否定できません。

## 検索ノイズになるかも

社員が気軽に閲覧しやすい状態にしたいため、普段使うチャット上に投稿することに意味があると考えていますが、チャット全体を検索した時に記事が引っかかることがあります。

問題解決への助けになることもあると思いますが、これを検索ノイズとして捉える場合、古い投稿を削除するバッチ処理を作ることで解決できるかも知れません。
ただ Chatwork の公開 API ではチャンネル内の過去メッセージを一括削除することは難しいのですが…

# 最後に

懸念点もある通り、導入に関しては賛否両論あるかと思いますが、私の会社では上司や他の社員から好評を得ることができました。

実はこの度退職することとなったのですが、RSS チャンネルに関しては要望があり、他のメンバーに運用方法など引き継ぎを行いました。
私が去った後も元気に稼働し続けてくれることを願っています。

また、この記事を書くにあたって他社や組織で同じような事例があるか軽くググってみましたが、なかなか見つかりませんでした。
個人が Slack で RSS を購読して情報収集するのはよくある使い方ですが、組織など大きな単位での事例はあまりないのかなと思いました。
もしくは当たり前過ぎて記事にするほどでもなかったり…？

パッケージ化して効果測定なども行えるようにすれば法人向けにワンチャンありそうじゃないですかね？
新規事業の可能性を感じた企業/個人様がいらっしゃいましたらお力添えいたしますのでお声かけください。