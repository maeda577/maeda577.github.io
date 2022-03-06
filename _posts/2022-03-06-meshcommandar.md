---
layout: post
title: MeshCommanderをつくるDockerfile
date: 2022-03-06 20:08 +0900
---
vPro対応のPCを買ったものの公式ツールが使えなかったのでOSSの方をコンテナ化しました

## 前提条件

* vPro側PCはHP EliteDesk 800 G2 DM
* クライアント側OSはMacでDocker Desktopを使う
    * Windowsがあれば普通にIntel Manageability Commanderを使えば良い気がする
* KVM機能を使う場合、vPro側PCにディスプレイを繋いでおかないと画面が出ない
* Serial over LAN機能はうまく動かなかったので諦める
    * 上物OSのログイン画面は出るが文字が入らない

## MeshCommandarコンテナ作成

* こんな感じでDockerfileを作る
    * [MeshCommanderのコンテナを作るDockerfile](https://gist.github.com/maeda577/d33616338cd5dbca648e53b8aff42783)
* 以下コマンドでコンテナ作成。Linux系OSの場合は適宜sudoする
    * nodejsはPID1で動かすとよろしくないらしいので--initをつける
    * [docker-node/BestPractices.md at main · nodejs/docker-node · GitHub](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md#handling-kernel-signals)
* バージョンは以下にあるものを指定
    * [meshcommander - npm](https://www.npmjs.com/package/meshcommander)

``` shell
# バージョンは0.9.3-b
docker build --tag localhost/meshcommandar:0.9.3-b --build-arg VERSION=0.9.3-b https://gist.githubusercontent.com/maeda577/d33616338cd5dbca648e53b8aff42783/raw/8fe13e0adee33ebe83f0083f70f6980583ac0bbd/Dockerfile
docker run --name MeshCommandar --detach --init --publish 3000:3000 localhost/meshcommandar:0.9.3-b
```

* あとは `http://<Hostname or IPAddress>:3000/` にアクセスすればWebUIが開く
