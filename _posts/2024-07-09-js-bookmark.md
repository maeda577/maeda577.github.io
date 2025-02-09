---
layout: post
title: 全Webページの読み取り権限を求める系アドオンを置き換えるJavaScriptブックマーク
date: 2024-07-09 20:40 +0900
---

「既存アドオンがあるけど、全Webページの読み取り権限が必要で気持ち悪い...」系アドオンの代替を目指しました

## 前提条件

MacのSafariでテストしてます。他のブラウザで動作するかは不明です。

## 今見ているWebページをMarkdown形式でコピーする

* 調べると`execCommand`を使った複雑なものが多いが、今だと関数一発で行けた
* タイトルの`|`は`\|`に置き換える
* 参考: [文字列をコピーするブックマークレットで使うクリップボードのAPI改めて見てた - hogashi.*](https://blog.hog.as/entry/2021/09/30/021450)

``` javascript
javascript:navigator.clipboard.writeText(`[${document.title.replaceAll('|','\\|')}](${location.href})`)
```

## PocketにWebページを追加する

* 通常は公式の [Pocket: 保存方法](https://getpocket.com/add) にあるブックマークを使えばいい
* このブックマークはMacのSafariだと動かない
    * たぶん`サイト越えトラッキングを防ぐ`の設定が原因
* ほぼここのコピペ: [Pocket Bookmarklet That Avoids Cookie Errors](https://gist.github.com/kortina/8cbba68393606d1b5bfc3aa0c13eea36)

``` javascript
javascript:open('https://getpocket.com/edit?url='+encodeURIComponent(document.location.href),'pocketAdd','height=450,width=650,popup')
```
