---
layout: post
title: Github Pagesでjekyllのサイトを作ってWSLでテストする方法
date: 2019-11-04 22:51 +0900
---

n番煎じな内容ですが、とりあえず最低限の作業量でjekyllテスト環境を作ったりする手順です。Ruby未経験でbundlerとは何ぞやという所からスタートだったので結構ハマりました。

前提条件・構築方針
---------------------

- jekyllのビルドはGithub Pages上で動いているものを使う（HTMLのアップロードは行わない）
- 手元の表示テストにはWSL Ubuntuを使う
    - 2020/09/22追記 WSL2とUbuntu 20.04で構築するよう修正
- WSL上ではRubyを使っておらず、今後も使う予定がない
    - rbenvとかは考えない

Github上のリポジトリ作成
---------------------
何はともあれこれがないと始まらないので、以下URLを参考にリポジトリを作る。リポジトリ名は\<user>.github.ioにせよと書いてあるので従う。何でもいいっぽいけど、これに合わせるとURLがすっきりする。

[Creating a GitHub Pages site - GitHub ヘルプ](https://help.github.com/ja/github/working-with-github-pages/creating-a-github-pages-site)

WSL Ubuntuの環境構築
---------------------
まずはこのあたりを読みながらWSL2とUbuntu20.04を有効化する。

[Windows Subsystem for Linux (WSL) を Windows 10 にインストールする | Microsoft Docs](https://docs.microsoft.com/ja-jp/windows/wsl/install-win10)

インストール後はアップデートしRuby導入。後々sudoするのが大変になるらしいので、bundlerはユーザー領域に導入する。
``` shell
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install ruby
gem install bundler --user-install
```

bundlerのインストール時にパスが通っていないと言われるので通す。

[Frequently Asked Questions - RubyGems Guides](https://guides.rubygems.org/faqs/#user-install)

``` shell
vi ~/.bashrc
```
末尾に追加。
``` diff
@@ -116,3 +116,7 @@
   fi
 fi

+if which ruby >/dev/null && which gem >/dev/null; then
+  PATH="$(ruby -r rubygems -e 'puts Gem.user_dir')/bin:$PATH"
+fi
+
```

bashrcの再読み込み
``` shell
source ~/.bashrc
```

適当なディレクトリに移動してからリポジトリをcloneする。git cloneで「chmodが失敗した」といったエラーが出た場合はOS再起動で直るはず。
``` shell
cd /path/to/source/directory
git clone https://github.com/<user>/<user>.github.io.git
cd <user>.github.io
```

Gemfileを作成する。そこにGithub Pages用のgemファイルを追加してinstallする。依存パッケージが無いと怒られるので入れ、gemのインストール先もユーザー領域に指定する。

[github/pages-gem: A simple Ruby Gem to bootstrap dependencies for setting up and maintaining a local Jekyll environment in sync with GitHub Pages](https://github.com/github/pages-gem)
``` shell
bundle init
echo "gem 'github-pages', group: :jekyll_plugins" >> ./Gemfile
sudo apt-get install ruby-dev make gcc g++ zlib1g-dev
bundle install --path ~/.gem/
```

これでjekyllが動くはずなので新規サイトを作成する。既にファイルがあるのでforceをつける。上書きされるのはGemfileだけのはず。
``` shell
jekyll new . --force
```

新Gemfileファイルの真ん中らへんにgithub-pages用gemの定義があるのでコメントアウトを外す
``` shell
vi .Gemfile
```
``` diff
-#gem "github-pages", group: :jekyll_plugins
+gem "github-pages", group: :jekyll_plugins
```

一応。多分何も起きない
``` shell
bundle install
```

サイトを生成してみる。生成結果は [http://127.0.0.1:4000/](http://127.0.0.1:4000/) で見られる。
``` shell
jekyll serve
```

とりあえず表示できたら一旦gitに投げる。.bundleディレクトリにはホームディレクトリの名前とか入ってるので流石にgitから外す。
``` shell
echo ".bundle" >> .gitignore
git add --all
git status
git config user.email "hogehoge@domain.com"
git config user.name "testtest"
git commit -m "Initial commit"
git push -u origin master
```

[https://\<user>.github.io](https://\<user>.github.io) で表示できることを確認して完成！

追加で色々やること
---------------------

jekyllには「新規投稿を作る」といったコマンドが無く、個人的には欲しいのでjekyll composeを入れる。

[jekyll/jekyll-compose: Streamline your writing in Jekyll with these commands.](https://github.com/jekyll/jekyll-compose)
``` shell
echo "gem 'jekyll-compose', group: [:jekyll_plugins]" >> ./Gemfile
bundle install
```

faviconのエラーログを黙らせる。
``` shell
touch favicon.ico
```
それ以外にも下記が必要。

- about.mdを修正する
- _config.yamlを全体的に直す
- _posts内のwelcomeページは消す

テーマ設定
---------------------
GitHub Pagesでは以下のテーマを標準でサポートしている。

[Supported themes \| GitHub Pages](https://pages.github.com/themes/)

ただし、_layouts にdefaultしか定義していないものが多く、そのまま適用すると「Build Warning: Layout 'home' requested in index.md does not exist.」とか出てトップページが表示されなくなる。単一ページを作るだけなら問題ないが、ブログっぽいものを作るときは大変不便。

諸々悩んだ結果、Minimal Mistakesというテーマが良さそうなので適用した。[mmistakes/minimal-mistakes: Jekyll theme for building a personal site, blog, project documentation, or portfolio.](https://github.com/mmistakes/minimal-mistakes)

適用するには_config.yamlや色々リソースを作る必要があるため、以下のスターターからファイルをコピーしてくると大変楽。

[mmistakes/mm-github-pages-starter: Minimal Mistakes GitHub Pages site starter](https://github.com/mmistakes/mm-github-pages-starter)

2020/06/06追記

トップページが出ないのはindex.htmlを作成していないため。作ったらいい感じになったので、テーマをGitHub Pagesで標準サポートしているCaymanに変更しました。合わせて_layoutsにページ定義を作成していくとカスタマイズしやすくて良き。

2020/07/25追記

結局テーマをminimaに戻しました。

その他
---------------------

GitHub Pagesで動いているプラグインなど。これらは_config.yamlのpluginsで使っても動くはず。

[Dependency versions \| GitHub Pages](https://pages.github.com/versions/)
