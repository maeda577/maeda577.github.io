---
layout: post
title: Windows ServerバックアップでESXiのRDMディスクへのバックアップが失敗する時は仮想互換モードにすると直るかもしれない
date: 2020-09-05 23:21 +0900
---
ESXi上のWindowsServerでバックアップが失敗してた時の顛末です。

前提条件
---------------------
* ESXi 7.0上にWindows Server 2019 Standardが立っている
* 物理ディスクをRDMでWinServerに認識させている
* RDMしているHDDは2TBと3TBが1本ずつ

何が起きたか
---------------------
* Windows Serverバックアップを実行すると失敗する
* エラーメッセージは「Windows バックアップで保存場所にシャドウコピーを作成できませんでした。」
* イベントビューアーを見ると以下のログが出ている

```
'開始したバックアップ操作は、次のエラー コード '0x80780034' (Windows バックアップで保存場所にシャドウ コピーを作成できませんでした。) のため失敗しました。イベントの詳細で解決策を確認し、問題の解決後にバックアップ操作を再実行してください。
```

```
ボリューム シャドウ コピー サービス エラー: 予期しないエラー DeviceIoControl(\\?\Volume{a59dcd66-5963-4ba8-995c-915935cb8654} - 000000000000024C,0x0053c008,00000219B0C80150,0,00000219B0C81160,4096,[0]) です。hr = 0x80070057, パラメーターが間違っています。
。 

操作:
   EndPrepareSnapshots を処理しています

コンテキスト:
   実行コンテキスト: System Provider
```
パラメーターが間違っています。と言われましても…

調査
---------------------
* 色々ググった結果以下URLを見つける
    * [Windows10でバックアップ失敗 - 「シャドウコピーを作成できませんでした。」 : The Way to Nowhere](http://blog.ntakano75.com/archives/68666292.html)
* 「ディスクの管理」で見える容量と「diskpart」で見える容量が違うらしいのでやってみる
![image](/assets/images/2020/winbackup.jpg)
Disk1の容量がまるで違う…何故…Disk2は合ってる…

対応
---------------------
* 上の記事の通りSATAコントローラーのドライバを更新してみる
    * 「既に最新です」と言われ更新できず
* RDMを仮想互換モードで作り直してみる
    * [RDM の仮想および物理互換モード](https://docs.vmware.com/jp/VMware-vSphere/7.0/com.vmware.vsphere.storage.doc/GUID-4B2479B1-541D-4FF4-865E-2EE711294478.html) を見ると最近は2TB超えも問題ないらしい
    * [仮想互換モードの Raw デバイス マッピングの作成](https://docs.vmware.com/jp/VMware-vSphere/7.0/com.vmware.vsphere.storage.doc/GUID-43684552-655C-4893-AE02-9992F51D87B6.html) の通り、-rオプションでRDMを作り直す
    * 直った！！！！！！！

コマンドサンプルは以下

``` shell
# 物理デバイスの一覧。必要なのはConsole Deviceの列
esxcfg-scsidevs --compact-list
# 仮想互換モードでRDMを作る。作成先は既存のdatastore
vmkfstools --createrdm /vmfs/devices/disks/t10.ATA_____TOSHIBA_MQ04ABD200_________________________________<シリアルっぽい文字列> /vmfs/volumes/datastore1/rdm_TOSHIBA_MQ04ABD200.vmdk
```

おわりに
---------------------
* 2TB超えのディスクをRDMするときは仮想互換モードで作るべし
* 「パラメータが間違っています」ってエラーメッセージどうにかならないですかね…原因が追えない
