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
- WSL上ではRubyを使っておらず、今後も使う予定がない
    - rbenvとかは考えない

Github上のリポジトリ作成
---------------------
何はともあれこれがないと始まらないので、以下URLを参考にリポジトリを作る。リポジトリ名は\<user>.github.ioにせよと書いてあるので従う。何でもいいっぽいけど、これに合わせるとURLがすっきりする。

[Creating a GitHub Pages site - GitHub ヘルプ](https://help.github.com/ja/github/working-with-github-pages/creating-a-github-pages-site)

WSL Ubuntuの環境構築
---------------------
まずはこのあたりを読みながらWSLを有効化する。ディストリビューションはUbuntuで。

[Windows Subsystem for Linux を有効にし、ディストリビューションをインストールする - Learn \| Microsoft Docs](https://docs.microsoft.com/ja-jp/learn/modules/get-started-with-windows-subsystem-for-linux/2-enable-and-install)

インストール後はアップデートしRuby導入。後々sudoするのが大変になるらしいので、bundlerはユーザー領域に導入する。
``` shell
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install ruby2.5
gem install bundler --user-install
```

bundlerのインストール時にパスが通っていないと言われるので通す。

[Frequently Asked Questions - RubyGems Guides](https://guides.rubygems.org/faqs/#user-install)

``` shell
vi ~/.bashrc
```
末尾に以下を追加。
``` diff
+ # add PATH to user-install gems.
+ if which ruby >/dev/null && which gem >/dev/null; then
+   PATH="$(ruby -r rubygems -e 'puts Gem.user_dir')/bin:$PATH"
+ fi
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
sudo apt-get install ruby2.5-dev make gcc g++ zlib1g-dev
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
- #gem "github-pages", group: :jekyll_plugins
+ gem "github-pages", group: :jekyll_plugins
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

ただし、_layouts にdefaultしか定義していないものが多く、そのまま適用すると「Build Warning: Layout 'home' requested in index.md does not exist.」とか出てトップページが表示されなくなる。単一ページを作るだけなら問題ないが、ブログっぽいものを作るときは大変不便。自分で定義すればいいのだがHTMLがさっぱり分からないので、標準テーマの_layoutsをコピーしておく。

``` shell
mkdir _layouts
wget https://raw.githubusercontent.com/jekyll/minima/master/_layouts/home.html -O ./_layouts/home.html
wget https://raw.githubusercontent.com/jekyll/minima/master/_layouts/page.html -O ./_layouts/page.html
wget https://raw.githubusercontent.com/jekyll/minima/master/_layouts/post.html -O ./_layouts/post.html
```
コピーが済んだらいい感じのテーマを選んで適用する。
``` shell
vi ./_config.yaml
```
``` diff
- theme: minima
+ theme: jekyll-theme-cayman
```
とりあえず表示されるものの、レイアウトが最低限な感じでブログっぽくない。あと_includesとかもコピーした方がいい気がする。気合と根性で何とかするしかない。

その他
---------------------

GitHub Pagesで動いているプラグインなど。これらは_config.yamlのpluginsで使っても動くはず。

[Dependency versions \| GitHub Pages](https://pages.github.com/versions/)
