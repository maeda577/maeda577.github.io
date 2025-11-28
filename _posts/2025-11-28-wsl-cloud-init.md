---
layout: post
title: WSLのUbuntuをcloud-initで初期設定する
date: 2025-11-28 15:24 +0900
---
最近はWSLの初期設定も便利になってました

## 参考

* [第869回 WSLでもcloud-initを活用する](https://gihyo.jp/admin/serial/01/ubuntu-recipe/0869)

## cloud-init設定YAMLの作成

ファイル名によって対象ディストリビューションを変えられるが、今回は「全てのUbuntu」にした

``` powershell
# ディレクトリとファイルを作成する
New-Item -Type Directory -Path $HOME\.cloud-init\
New-Item -Path $HOME\.cloud-init\Ubuntu-all.user-data
# メモ帳を起動し、設定を入れていく。もちろんVSCode等でいじった方が良い
notepad.exe $HOME\.cloud-init\Ubuntu-all.user-data
```

* 使えるモジュールの一覧: [Module reference](https://cloudinit.readthedocs.io/en/latest/reference/modules.html)

``` yaml
#cloud-config

# ロケール
locale: ja_JP.UTF-8

# デフォルトユーザーの作成
users:
  - name: user01
    groups:
      - adm
      - sudo
      - users
    shell: /bin/bash
# パスワードの設定
chpasswd:
  expire: false
  users:
    - name: user01
      type: hash
      # パスワードはIDと同じ。この文字列は openssl passwd -6 で生成できる
      password: $6$9mufZUNSHSBfJYtg$MPP4BG9zgO2elU3oULw.TygvmQFAs41lTlz/VkhrhA4.fNOcxfafe2ITBZ4RP5vvl4CFX8JkbrevrkQk8VU4o1

# aptの参照先をICSCoE(IPAの産業サイバーセキュリティセンター)に向ける
# 気分の問題なので、変える必要が無ければaptアイテムをごそっと消す
apt:
  preserve_sources_list: false
  primary:
    - arches: [default]
      uri: https://ftp.udx.icscoe.jp/Linux/ubuntu/
  security:
    - arches: [default]
      uri: https://ftp.udx.icscoe.jp/Linux/ubuntu/

# apt update と apt upgrade する
package_update: true
package_upgrade: true

# podmanインストールする
packages:
  - podman

# 諸々の設定ファイルを書き込む
write_files:
  # WSLのデフォルトユーザーを明示する
  - path: /etc/wsl.conf
    content: |
      [user]
      default=user01
    append: true
    defer: true
```

## WSLセットアップ

``` powershell
# WSL本体のみをインストール 既に入っているなら不要
wsl --install --no-distribution

# インストール可能なディストリビューションを見てからインストール
wsl --list --online
wsl --install --no-launch Ubuntu-24.04

# 上のインストール方法だと毎回ダウンロードが走ってしまうため、cloud-initで試行錯誤する時は
# https://releases.ubuntu.com/ からWSL用イメージをダウンロードしておいてからインストールした方が良い
# wsl --install --no-launch --from-file $HOME\Downloads\ubuntu-24.04.3-wsl-amd64.wsl

# WSLディストリビューションを起動し、cloud-initが終わるまで待つ
wsl --distribution Ubuntu-24.04 --user root -- cloud-init status --wait
# 念のため一回停止する
wsl --terminate Ubuntu-24.04
# 以降は普通に起動して使う
wsl --distribution Ubuntu-24.04

# WSLディストリビューションを消すときはunregister
# wsl --unregister Ubuntu-24.04
```
