---
layout: post
title: XG Firewallでtransix使いつつPPPoEのサーバー立てようとして失敗した
date: 2020-08-08 13:59 +0900
---
[前回XG Firewallでtransix](/2020/07/24/sophos-transix.html)した目的がVPNサーバーを立てる事だったのですが失敗しました。

経緯
-----------------
* やりたいこと
    * 公衆無線LANで安全に接続するための自宅VPNサーバーを作る
        * どのグローバルIPから繋ぐか不定
        * FWの透過しやすさを考えSSL-VPNを使う
    * transixだとサーバー立てられないのでPPPoEで立てる
* やろうとしたこと
    * XG FirewallのSSL-VPNを有効化してPPPoEで待ち受ける
    * 通常の通信は全てtransixに流し、SSL-VPNの通信だけPPPoEに流す
* うまくいかなかったこと
    * スタティックルートで0.0.0.0/0をtransixに向けているため、VPNすると行きがPPPoEで帰りがtransixの非対称経路になり動かない
        * 非対称経路が原因という根拠はなく、transixの何かで止まっている可能性もあり
        * スタティックルートを消し全部PPPoE経由で流すとうまくいく
    * ポリシールートを書こうにも「トンネルに向ける」という設定が書けずtransixに流せない

どうするか
-----------------
諦めて仮想マシン2台立てました。transixをSEIL/x86で、PPPoEをXG Firewallで捌きます。
