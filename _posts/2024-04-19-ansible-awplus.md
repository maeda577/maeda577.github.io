---
layout: post
title: アライドテレシス AlliedWare Plus対応機器をAnsibleで管理する最低限の設定
date: 2024-04-19 02:06 +0900
---

AT-x230-10GT を買ったのでAnsible管理しようと思ったものの、サンプルコードが少なすぎたので書きました。

## 専用モジュールのドキュメント

* 公式ページなどは無いので、AnsibleGalaxyのdocsを読むしかない
  * [Ansible Galaxy - alliedtelesis.awplus](https://galaxy.ansible.com/ui/repo/published/alliedtelesis/awplus/docs/)
* Githubのリポジトリもあるが、さほど情報は無い
  * [alliedtelesis/ansible_awplus: Ansible Network Collection for AlliedWare Plus devices](https://github.com/alliedtelesis/ansible_awplus)

## 下準備

Dockerの `docker.io/library/python:bookworm` を使っているので、最低限のPython3は入っている前提

``` shell
# ansible-pylibsshのビルドに必要なパッケージ
apt update --yes
apt install libssh-dev --yes

# Ansible本体とnetwork_cli用のSSHモジュール
# 以前はparamikoが使われていたが、最近はansible-pylibsshを使うらしい
pip3 install ansible-core ansible-pylibssh
# awplusモジュールはnetwork_cliに依存しているのでansible.netcommonも入れる
ansible-galaxy collection install ansible.netcommon alliedtelesis.awplus
```

## 最低限のPlaybook本体

``` yaml
- hosts: x230
  tasks:
    # お試しでホスト名を設定する
    - name: Set hostname.
      alliedtelesis.awplus.awplus_system:
        hostname: x230
      become: true  # cli操作で言う所のenableに相当する

    # モジュールが設定するのはrunning-configなので、随時保存する
    - name: Save config.
      alliedtelesis.awplus.awplus_config:
        save_when: modified   # running-configとstartup-configに差分があれば保存
      become: true
  # awplusモジュールはnetwork_cliのconnectionが必須
  connection: network_cli
  # このあたりは普通inventoryで指定する
  vars:
    # OSは普通自動で検知してくれるらしい。検知してくれなかったのでFQDNで指定するとちゃんと動く
    ansible_network_os: alliedtelesis.awplus.awplus 
    ansible_user: manager
    ansible_host: 192.168.11.22
```

## おわりに

ググってもサンプルほとんど出なかったので誰も使ってない説あった
