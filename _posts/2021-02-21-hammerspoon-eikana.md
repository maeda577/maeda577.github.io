---
layout: post
title: Hammerspoonを使いmac上で半角/全角キーを動作させる
date: 2021-02-21 00:41 +0900
---
自宅ではMac、職場ではWindowsでどうしても半角/全角キーを動かしたかったのでやりました。

前提条件
----------------
* MacにWindows用日本語キーボードを繋げていること
* Hammerspoonがインストールされていること

必要なconfig
----------------
結果的にこれで動きました。
``` lua
hs.hotkey.bind({}, "`", function()
    if hs.keycodes.currentMethod() == "Hiragana" then
        hs.eventtap.keyStroke({}, 0x66) -- 英数キー
    else
        hs.eventtap.keyStroke({}, 0x68) -- かなキー
    end
end)
```

1. Macだと半角/全角キーが`になるので、単体で(修飾キー無しで)それを押されたことを置換する
    * @ キーの上にある`はHammerspoon的には「Shift+@」なので普通に入力できる
1. 現在の入力メソッドを調べる
    * 英数モードの時はnilになってしまうので、かな入力モードを判定する
1. 今かな入力ならば英数キーを押し、英数モードならかなキーを押す
