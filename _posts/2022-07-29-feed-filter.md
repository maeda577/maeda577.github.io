---
layout: post
title: RSS/Atomフィードから不要な要素をXPathで除外する簡易APIを自作した
date: 2022-07-29 20:36 +0900
---

RSSフィードの絞り込みをやりたくて自作しました。

成果物
-------------------
[maeda577/RESTfulFeedFilter: RSS/Atom feed filtering API using XPath](https://github.com/maeda577/RESTfulFeedFilter)

* 機能
    * RSS/Atomフィードを読み込み、指定された要素を除外したフィードを生成するAPI
* デモサイト
    * [https://rff.azurewebsites.net/](https://rff.azurewebsites.net/)

発端
--------------
* ニュースサイトのRSSを購読したい
* 大手ニュースサイトでは「ニュース全て」と「カテゴリ別」のRSSが用意されているが、中堅ぐらいのサイトだと「全て」しかない事がある
    * たとえばYahoo!ニュースでは[カテゴリ別のRSS一覧](https://news.yahoo.co.jp/rss)があるが、全てのニュースサイトがこうとは限らない
* 「全て」では多すぎるので絞り込みたい
* RSSはXMLなんだからXPath使えるじゃんと思いつく

事前調査
--------------
似たようなサービスは既にある

* [RSSフィードフィルタ](https://syon-feed-filter.herokuapp.com/)
    * タイトルに含まれるキーワードやリンクのドメインなどで絞り込める
* [RSS Feed Generator, Create RSS feeds from URL](https://rss.app/)
    * いろいろ絞り込める。無償プランではフィルタ無しっぽい
    * [[Updated] How to Filter RSS Feeds in 2021](https://rss.app/blog/how-to-filter-rss-feeds-JC9I1k)
* [Zapier \| Automation that moves you forward](https://zapier.com/)
    * これも無償プランでは絞り込めなさそう
* 他にもRSSリーダ自体にフィルタ機能があったりしたが、総じて有償プランだった
* XPathで絞り込むようなサービスは見つからなかった

実装する環境の選定
--------------
* Inoreaderを普段使いしているのでグローバルからアクセス出来ることが必須
* 最近のクラウドサービスは無償プランが結構気前良いので無課金で作る
* 最初Cloudflare Workersで作ろうとしたがDOM周りの関数が無いようなので手を出さなかった
* 結局慣れているAzure App Serviceと.netで作ることにした
    * 最終的にDockerコンテナに固めたのでAzureである必要は無くなった

実装
--------------
* コード周り
    * 近代的なASP.NET Core 6で作り、画面も最近流行っている感じのSwaggerで生成する
    * XML操作部分は普通にXmlDocument使う
    * [プレフィックス無しのXPathは「Namespace無し」と解釈される仕様のため](https://docs.microsoft.com/ja-jp/dotnet/api/system.xml.xmlnode.selectnodes?view=net-6.0#remarks)、デフォルトxmlnsがあれば適当にXPath側にコピーしておく
* デプロイ周り
    * DockerコンテナをAzureAppServiceにデプロイする場合、[80番か8080番で待ち受ける](https://docs.microsoft.com/ja-jp/azure/app-service/configure-custom-container#configure-port-number)前提になっている
        * ASP.NETのWebAPIテンプレートだとHTTPSに強制リダイレクトされるようになっているので切っておく
    * AzureAppServiceではデフォルトで WEBSITES_ENABLE_APP_SERVICE_STORAGE がfalseになっていて静的ファイルの公開が出来ないので忘れずtrueにする

既知の不具合
--------------
* FeedlyでRSS1.0フィードの絞り込みをかけた場合認識しない
    * Feedly側でエラーメッセージが出ないので何もわからない

終わりに
--------------
XPathのnamespace周りでハマり倒したので辛さあった
