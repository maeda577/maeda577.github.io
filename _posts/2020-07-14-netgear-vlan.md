---
layout: post
title: NETGEARの拡張802.1Q VLAN設定とCisco用語の対応表
date: 2020-07-14 23:59 +0900
---
アンマネージプラススイッチを買ったもののVLAN用語がいまいちピンと来なかったので調べました。

前提条件
-------------------------------
* 機器はNETGEAR GS308E
    * 8ポートのアンマネージプラススイッチ
    * WebGUIのみでコンソールやSSHなどのCUI操作は不可
* ファームウェアバージョンは V1.00.05JP
* 使うのは拡張802.1Q VLAN
    * ポートベースVLANや基本802.1Q VLANは不明

VLAN用語の対応表
-------------------------------

多分こんな感じだと思う。

| NETGEAR | Cisco | メモ |
| -- | -- | -- |
| 拡張802.1Q VLANを [有効] | 全ポートで <br /> switchport mode trunk <br /> switchport trunk native vlan 1 <br /> switchport trunk allowed vlan 1 |
| VLANポートメンバーで \<VLAN ID\> を [追加] | vlan \<VLAN ID\> | 必須 |
| VLANメンバーシップでポートの \<VLAN ID\> を [T] にする | switchport trunk allowed vlan add \<VLAN ID\> | 
| VLANメンバーシップでポートの \<VLAN ID\> を [U] にする| (相当するものなし。強いて言えばnative vlanとして指定するためだけのallowed vlan add) | これを指定しないと<br />下のPVIDを指定できない |
| VLANメンバーシップでポートの \<VLAN ID\> を指定なしにする| switchport trunk allowed vlan remove \<VLAN ID\> |
| PVID設定で \<VLAN ID\> を [適用] | switchport trunk native vlan \<VLAN ID\> | 
