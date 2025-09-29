記事を書くときに使うコマンドのメモ
------------------------------------
要jekyll-compose

```shell
# 新しい下書きを作成
jekyll draft hogehoge

# 下書きを含めてテストサーバ起動
jekyll serve --drafts

# 下書きを公開
jekyll publish _drafts/hogehoge.md

# 公開したものをひっこめる
jekyll unpublish _posts/yyyy-mm-dd-hogehoge.md
```


