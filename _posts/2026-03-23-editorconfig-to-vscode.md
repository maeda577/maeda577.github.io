---
layout: post
title: EditorConfigっぽいことをVSCodeの標準機能でやる
date: 2026-03-23 19:54 +0900
---

VSCodeの標準機能でもそれなりに何とかなりました

## EditorConfigとVSCodeの設定対応表

| 設定内容 | EditorConfig | VSCode |
|---|---|---|
| インデントのTab/Space切り替え | `indent_style` | `editor.insertSpaces` |
| インデントの幅 | `indent_size` | `editor.tabSize` |
| 改行コードの設定 | `end_of_line` | `files.eol` |
| 文字コードの設定 | `charset` | `files.encoding` |
| 各行末尾のスペースを消す | `trim_trailing_whitespace` | `files.trimTrailingWhitespace` |
| ファイルの末尾に改行を入れる | `insert_final_newline` | `files.insertFinalNewline` |
| ファイルの末尾の余計な改行を消す | - | `files.trimFinalNewlines` |

### VSCode側の注意点

* `editor.detectIndentation`を無効化しないと既存のインデントに合わせてしまう
* `files.eol`と`files.encoding`はあくまでデフォルト値を設定するだけなので、既存ファイルの設定は置き換えられない

## VSCode設定サンプル

``` jsonc
{
    // インデントの検知を切る
    "editor.detectIndentation": false,
    // Tabの代わりにSpaceを使う
    "editor.insertSpaces": true,
    // Tabの幅 実際はSpaceの個数
    "editor.tabSize": 2,
    // デフォルトの改行コード あくまでデフォルトなので既にCRLFなファイルには効かない
    "files.eol": "\n",
    // デフォルトの文字コード あくまでデフォルトなので既にあるファイルには効かない
    "files.encoding": "utf8",
    // 末尾に改行を足す
    "files.insertFinalNewline": true,
    // 末尾のよけいな改行を消す
    "files.trimFinalNewlines": true,
    // 各行の末尾のスペースを消す
    "files.trimTrailingWhitespace": true,

    // 言語ごとに設定を変えることもできる。EditorConfigのようにファイルパスでの指定は出来なさそう
    "[typescript]": {
        "editor.tabSize": 4,
    },

    // ファイルの保存時に自動でフォーマットをかける。好みの分かれる設定だと思う
    "editor.formatOnSave": true,
}
```
