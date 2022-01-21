記事を書くときに使うコマンドのメモ
------------------------------------
要jekyll-compose

```shell
# 新しい下書きを作成
bundle exec jekyll draft hogehoge

# 下書きを含めてテストサーバ起動
bundle exec jekyll serve --drafts

# 下書きを公開
bundle exec jekyll publish _drafts/hogehoge.md

# 公開したものをひっこめる
bundle exec jekyll unpublish _posts/yyyy-mm-dd-hogehoge.md
```


