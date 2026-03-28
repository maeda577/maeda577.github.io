---
layout: post
title: NetworkManagerのメモ
date: 2026-03-29 00:44 +0900
---
NetworkManagerが覚えられないので備忘録です。

## 前提条件

OSはRHEL10です
```
[redhat@rhel ~]$ cat /etc/redhat-release
Red Hat Enterprise Linux release 10.1 (Coughlan)
[redhat@rhel ~]$
```

## バイナリ

* 本体は `NetworkManager` でsystemd経由で起動されている
* 操作用のコマンドは `nmcli`
    * `nmtui` もあり、後者の方が若干グラフィカルに操作できるが出来ることが少ない
    * 他にもcockpit経由など、いろいろいじる方法はある

## configファイルの置き場所

[NetworkManager.conf: NetworkManager Reference Manual](https://networkmanager.dev/docs/api/latest/NetworkManager.conf.html) によると以下っぽい

1. /etc/NetworkManager/
    * 普通のconfig置き場。手で書いたりするところ
1. /usr/lib/NetworkManager/
    * パッケージマネージャ経由で置く場所。chronyのdispacherとか入っている
1. /var/lib/NetworkManager/
    * /var なので動的に書き換えられる場所のはず。D-Bus経由とか？らしいがよく分かっていない
1. /run/NetworkManager/
    * /run なのでOS再起動で消える場所。設定が都度生成されるような場所で、手で書き換えることはない

NetworkManager.confを繋げた結果は `sudo NetworkManager --print-config` で見られる

## NICの設定ファイルの置き場所

RHEL10のインストール中にオンボードNICの設定を入れた時に置かれる設定

1. /etc/NetworkManager/system-connections/enp1s0.nmconnection
1. /run/NetworkManager/system-connections/lo.nmconnection

2個目はrunの下なので、OS起動に合わせて都度生成されているはずだが詳細は不明

## NICの設定ファイルの書き方

* toml形式で書く。パラメータはnmcliと同じらしい
    * [nm-settings-nmcli: NetworkManager Reference Manual](https://networkmanager.dev/docs/api/latest/nm-settings-nmcli.html)
* 反映は `sudo systemctl restart NetworkManager.service`
    * reloadも出来るはずだが、動きがよく分からなかったので再起動している
