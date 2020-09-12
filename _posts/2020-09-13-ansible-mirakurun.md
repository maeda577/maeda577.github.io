---
layout: post
title: MirakurunをAnsibleで構築した
date: 2020-09-13 00:23 +0900
---
先日やった[Mirakurun+Chinachuの変則的な構成](/2020/08/23/s270.html)のMirakurun部分をAnsible化しました。

やったこと
----------------------------
以下リポジトリの通りです。Ubuntu限定ですが、とりあえず投げればMirakurunが動くところまで行くと思います。

[maeda577/ansible-mirakurun: Mirakurun構築用Ansible Playbook](https://github.com/maeda577/ansible-mirakurun)

ハマったこと
----------------------------
* チャンネルスキャンがどうにも失敗する
    * 直前にMirakurunを再起動していたため、起動が終わらないうちにPUTを投げていた
    * uriモジュールが成功したらchangedになるだろうと思ってuntilを書いていたら常にokを返してた

おわりに
----------------------------
* 初めてAnsible使ってみたけど大変便利
