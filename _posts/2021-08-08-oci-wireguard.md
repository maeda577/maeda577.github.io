---
layout: post
title: Oracle Cloud Infrastructureで0円WireGuardサーバを立てる
date: 2021-08-08 23:11 +0900
---
Oracle Cloud InfrastructureのAlways Freeが大盤振る舞いだったのでWireGuardサーバを立てました。

前提条件
-------------------
* OCIのAlways Freeの範囲内で作る
* クライアントはAndroid端末
    * 公衆無線LANなどを使う想定なので、NAPT配下でかつNAPT後のグローバルIPアドレスは不定
* クライアントはこのサーバでNAPTしてインターネットに出られるようにする
    * Always FreeではVCNのNATゲートウェイが作れなかったので、ここでNAPTするのがよさそう
* WireGuardサーバは123/udpで待ち受ける
    * 公衆無線LANでも開いてそうだったから

IPアドレス関連のパラメータ
-------------------
* OCIのVCNに割り当てるセグメントは172.16.0.0/16
    * パブリックサブネットは172.16.90.0/24
    * プライベートサブネットは172.16.20.0/24
    * なんでもいい
* WireGuardでつくるセグメントは172.17.10.0/24
    * サーバは172.17.10.1
    * クライアントは172.17.10.10
* VCNのセグメントとWireGuardのセグメントが被るとOCIのルート表にルートが足せなくなるのでずらす
    * とはいえ今回の構成なら被っても問題はない

VCN作成
-------------------
まずはウィザードに沿って作る

* VCNウィザードは「インターネット接続性を持つVCNの作成」で作る。以下ができるはず
    * サブネット2個
    * ルート表2個
    * セキュリティリスト2個
    * インターネットゲートウェイ
    * DHCPオプション
* [セキュリティリスト]->[Default Security List]->[イングレスルールの追加] で以下を加える
    * ソースCIDR: 0.0.0.0/0
    * IPプロトコル: UDP
    * 宛先ポート範囲: 123

仮想マシン作成
-------------------
作成する際に「容量が不足しています。 後で再試行してください。」と言われるが根気よくやっていればいつか作れる

* [コンピュート]->[インスタンス]->[インスタンスの作成]
    * イメージはCanonical Ubuntu 20.04 (minimalじゃない普通の方)
    * シェイプはVM.Standard.A1.Flex
    * [パブリックIPv4アドレスの割当て]を選ぶ(デフォルトで選ばれている)

``` shell
# いつもの更新
sudo apt update
sudo apt upgrade
# タイムゾーン設定
sudo timedatectl set-timezone Asia/Tokyo
# HWEでWireGuardがマージされた以降のKernelに更新する(多分必須ではない)
# x86版のイメージでは最初から5.8.0だったので特に要らない
sudo apt install linux-generic-hwe-20.04
sudo reboot
# Kernelが5.6以降になっていればWireGuardがマージされているはず
hostnamectl
```

WireGuardサーバ構築
-------------------
* WireGuardはホストの数だけ鍵ペアが必要なので2組つくる
    * [Quick Start - WireGuard](https://www.wireguard.com/quickstart/)
    * Androidアプリ側で鍵を作ってもいい
* UbuntuなのでWireGuard設定もNetplan経由で行う
    * [Netplan \| Backend-agnostic network configuration in YAML](https://netplan.io/reference/)

``` shell
# 既存iptablesで123/udpを開ける
sudo iptables --insert INPUT 1 --protocol udp --dport 123 --jump ACCEPT
# ルール保存
sudo iptables-save --file /etc/iptables/rules.v4

# 周辺ツールも含めインストール
sudo apt install wireguard

# 作られたwireguardディレクトリのパーミッションを緩める(元は700)
sudo chmod 755 /etc/wireguard/
# 秘密鍵用の空ファイル作ってパーミッション設定しておく
sudo touch /etc/wireguard/server.key
sudo chmod 640 /etc/wireguard/server.key
sudo chown root:systemd-network /etc/wireguard/server.key
# クライアント用は自分だけが読めればいい
sudo touch /etc/wireguard/android.key
sudo chmod 600 /etc/wireguard/android.key

# サーバー用の鍵生成。.keyが秘密鍵で.pubが公開鍵
wg genkey | sudo tee /etc/wireguard/server.key | wg pubkey | sudo tee /etc/wireguard/server.pub
# クライアント用も生成
wg genkey | sudo tee /etc/wireguard/android.key | wg pubkey | sudo tee /etc/wireguard/android.pub

# Netplan設定
sudo vi /etc/netplan/90-network.yml
```

``` yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
      match:
        macaddress: xx:xx:xx:xx:xx:xx # 既存のnetplanのyamlからコピペ
      set-name: enp0s3
  tunnels:
    wg0:
      mode: wireguard
      addresses:
        - 172.17.10.1/24
      port: 123
      keys:
        private: /etc/wireguard/server.key # ホストの秘密鍵。自分しか触らないサーバなら直接書いてもいい
      peers:
        - allowed-ips: # Allowed IPsは「どの宛先へのトラフィックをこのpeerに流すか」のイメージ
            - 172.17.10.10/32
          keys:
            public: rlbInAj0qV69CysWPQY7KEBnKxpYCpaWqOs/dLevdWc= # android.pubの中身 これはファイル指定不可
  version: 2
```

``` shell
# 適用し問題なければEnter
sudo netplan try --timeout 10
# WireGuardのinterfaceが増えているはず
sudo wg show all
# うまく設定が反映されない時はnetworkdで以下が出ているかもしれない
# wg0: netdev exists, using existing without changing its parameters 
systemctl status systemd-networkd.service
# この場合の反映方法が分からなかったので不調だったら再起動
sudo reboot
```

WireGuardクライアント設定
-------------------
* クライアント用QRコードを生成して読ませる
    * 無理にサーバ側でやる必要もない
* configに関する公式ドキュメントはmanぐらいしかなかったので頑張って読む
    * [wireguard-tools - Required tools for WireGuard, such as wg(8) and wg-quick(8)](https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8)

``` shell
sudo vi /etc/wireguard/android.conf
```
``` ini
[Interface]
PrivateKey = </etc/wireguard/android.keyの中身>
Address = 172.16.30.10
DNS = 8.8.8.8   # クライアント側でDNSサーバをDHCPからもらう設定にしていて、それがローカルIPアドレスだとつながらなくなるのでDNSを固定させる
ExcludedApplications = com.google.android.gms   # GMSをVPNの対象外にする。VPNを張った状態でGoogleCastが出来ないためで、Castを使わないなら不要
 
[Peer]
PublicKey = </etc/wireguard/server.pubの中身>
EndPoint = <OCI上で確認できるホストのグローバルIP>:123
AllowedIPs = 0.0.0.0/0      # 公衆無線LANで使う想定なので全通信をWireGuard経由にする
PersistentKeepAlive = 30    # NAT配下、かつ一方的にパケットを受け取る場合は入れたほうがいいらしい。多分要らない
```
``` shell
# QRコード生成ツール入れる
sudo apt install qrencode
# コンソールにQRコードを出す。WindowsTerminalだと綺麗に出た
sudo qrencode --read-from=/etc/wireguard/android.conf --type=ansiutf8
```

* 出てきたQRコードをAndroidアプリのWireGuardで読み込む
* pingは通るはずなのでサーバからクライアントに向けてpingしてみる
    * インターネットには出られないはずなので疎通確認したら一旦切る

ルーティングの有効化
-------------------
* WireGuard経由で外に出られるような設定

``` shell
# パケットフォワーディングを有効化する
sudo sysctl --write net.ipv4.ip_forward=1
sudo sysctl --load

# iptablesでフォワードを全許可する
sudo iptables --insert FORWARD 1 --table filter --jump ACCEPT
# 外への通信でNAPTを有効化する
sudo iptables --insert POSTROUTING 1 --table nat --out-interface enp0s3 --jump MASQUERADE
# ルール保存
sudo iptables-save --file /etc/iptables/rules.v4
```

* もう一回Androidでつないでインターネットに出られることを確認する
    * ダメだったらサーバ側を再起動してみる

その他
-------------------
* ローカルIPアドレス宛の通信はNAPTしない設定
    * やるべきかどうかは完全に好み
* やる前にOCI側で以下をやっておく
    * OCIのルート表に「宛先を172.17.10.0/24、ターゲットをWireGuardのローカルIPアドレス」で加える
    * 仮想マシンのVNICの編集から「ソース/宛先チェックのスキップ」を有効にする

``` shell
sudo iptables --insert POSTROUTING 1 --table nat --destination 192.168.0.0/16 --jump ACCEPT
sudo iptables --insert POSTROUTING 1 --table nat --destination 172.16.0.0/12 --jump ACCEPT
sudo iptables --insert POSTROUTING 1 --table nat --destination 10.0.0.0/8 --jump ACCEPT
sudo iptables-save --file /etc/iptables/rules.v4
```
