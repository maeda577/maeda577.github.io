---
title: 「EACからGracenoteへ接続するためのCDDBサーバ」を停止しました
categories:
- Blog
date: 2019-11-28 22:02 +0900
---
色々あってGracenoteのAPIに繋げられなくなったので止めました。南無。

自作した [EACからGracenoteへ接続するためのCDDBサーバ](http://gncddb.azurewebsites.net/) の話です。

経緯
-----------------------

2019/11/26 CDの取り込みをしようとしたら楽曲情報が取得できない事に気付く。Client IDの更新忘れっぽいのでアプリを一度削除し、再度作成しようとしたら

> Currently unavailable, please check back later.

のエラー。詰む。

2019/11/27 [gracenote API に繋がらなくなったときに行う手段 : 犬ターネット](https://mgng.mugbum.info/1385) を見てLicense Renewal Formに入力する。解決せず。

2019/11/28 一日待ったらワンチャンあるのでは？と思ったが解決せず。いっそアカウント作り直そうとしたらアカウント新規作成フォームが無くなっていることに気付く。詰む。

もうだめだ！！！

これから
-----------------------
フォーラムを見ると、2018年2月ぐらいから発生しているようなので直す気なさそうな感じです。

[I can not add any app \| Gracenote Developer Music + Auto APIs](https://developer.gracenote.com/i-can-not-add-any-app)

現状としては

- 新規アプリ作成が動作しない
- 新規アカウント作成ができない
- 開発者ページからフォーラムへのリンクも無い
- 公式Twitterの更新は2017年3月で止まっている

と散々なので、もうGracenoteはAPI提供を辞めたいんだと思います。

という訳で、GracenoteのAPIが復活するまでは「EACからGracenoteへ接続するためのCDDBサーバ」を停止します。ご利用ありがとうございました！
