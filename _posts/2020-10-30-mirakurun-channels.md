---
layout: post
title: recfsusb2n.confとMirakurunのchannels.ymlを手で作る(地上波/BS)
date: 2020-10-30 23:28 +0900
---
ググるといくらでもサンプルが出てきますが、中身を理解するため自前で作りました。我が家はCSが入らないっぽいので除外してます。

はじめに
---------------------
BS部分のconfigを生成するExcelシートを作りました。これでBSのrecfsusb2n.confとchannels.ymlが作れます。地上波は居住地に沿ってよしなに作ってください。
* [BS_Channels.xlsx - Microsoft Excel Online](https://1drv.ms/x/s!AsyRfEh_oDaBzmlU00vgHoC9DC5N?e=s0GiDx)

チャンネル指定に必要な情報たち(地上波)
---------------------
地上波はマスプロ電工のサイトを見れば分かる。
* [地上デジタル放送　チャンネル一覧表 - マスプロ電工｜MASPRO](https://www.maspro.co.jp/contact/bro/bro_ch.html)
    * 地域ごとのチャンネル番号があり、地デジの場合はこの番号をchannelとして指定する

チャンネル指定に必要な情報たち(BS)
---------------------
地上波に比べBSは大変にしんどい。
* [BSデジタル放送局一覧 \| 一般社団法人放送サービス高度化推進協会（A-PAB）](https://www.apab.or.jp/bs/station/)
    * ここに書かれた「チャンネル番号」は論理チャンネルと呼ばれるらしく、各configに指定する際のSID(Service ID)に相当する
    * 物理のchannelはまた別
* [総務省｜令和2年版 情報通信白書｜事業者数及び放送サービスの提供状況](https://www.soumu.go.jp/johotsusintokei/whitepaper/ja/r02/html/nd251820.html)
    * 「図表5-1-8-8　BS放送のテレビ番組のチャンネル配列図」が物理チャンネルの一覧 (2021/03/13追記 リンク切れしてたのを修正)
        * 左旋は主に4K用なのでとりあえず気にしない
    * 1つの物理チャンネルに複数の放送が含まれている
* [茂木 和洋さんはTwitterを使っています 「ARIB TR-B15 記載のルールに従うと、明日からの NHK BS プレミアムの TSID は 0x4031 (16433) になるのだろうなっと。 https://t.co/kTuCCs3l4L」 / Twitter](https://twitter.com/kzmogi/status/993345535565156353)
    * TSID(Transport Stream ID)の求め方の抜粋。~~こんなん分かるか~~
    * 元のPDFは以下で246ページに記載あり。日本語版は有償なので、無償の英語版を参照
        * [OPERATIONAL GUIDELINE FOR DIGITAL SATELLITE BROACASTING ARIB TECHNICAL REPORT](http://www.arib.or.jp/english/html/overview/doc/8-TR-B15v4_6-3p4-E1.pdf)
* [リモート視聴要件と届出について \| 一般社団法人放送サービス高度化推進協会（A-PAB）](https://www.apab.or.jp/remote-viewing/implementation/)
    * TSIDを求める時にnetwork_idが必要で、BSのnetwork_idは一律でどのチャンネルでも4らしい

channels.ymlとは
---------------------
* mirakurunのチャンネル定義ファイル
* 書式と公式のサンプルは以下URLにある
    * [Mirakurun/Configuration.md at master · Chinachu/Mirakurun](https://github.com/Chinachu/Mirakurun/blob/master/doc/Configuration.md#channelsyml)
    * [Mirakurun/channels.yml at master · Chinachu/Mirakurun](https://github.com/Chinachu/Mirakurun/blob/master/config/channels.yml)
* channelパラメータはtuners.yamlに書いた録画コマンドにそのまま引き渡される

recfsusb2n.confとは
---------------------
* recfsusb2nのBS/CSチャンネル定義ファイル。地上波については何も書かなくていい
* チャンネル名、周波数、SID、TSIDを定義する
    * 周波数は100+物理チャンネル番号で指定する。1chなら101
        * 実際の周波数はrecfsusb2n内で計算される
        * CSだと200を足すっぽい(未検証)
    * SID(Service ID)はA-PABサイトの論理チャンネル番号
    * TSID(Transport Stream ID)はARIBの資料を元に頑張る
* channels.ymlに書いたchannelとrecfsusb2n.confのチャンネル名は必ず一致させる必要がある
    * 以下が参考になるが、結局はソース読む必要あり
        * [さんぱくん外出（US-3POUT）用のrecfsusb2nをC13～C63に対応させる - にゃののん日記](https://nyanonon.hatenablog.com/entry/20190607/1559917800)
* confのサンプルは以下
    * [recfsusb2n/recfsusb2n.conf at master · epgdatacapbon/recfsusb2n](https://github.com/epgdatacapbon/recfsusb2n/blob/master/src/recfsusb2n.conf)

地上波のチャンネル定義(channels.yml)
---------------------
これはシンプル。受信場所近くの放送塔からチャンネル番号を調べる。
* name: チャンネル名
* type: 地上波なのでGR
* channel: マスプロPDFに書かれているチャンネル番号(yamlではstringにするため'でくくる)

``` yaml
# 東京スカイツリーでのサンプル
- name: NHK 総合
  type: GR
  channel: '27'
- name: NHK Eテレ
  type: GR
  channel: '26'
```

地上波のチャンネル定義(recfsusb2n.conf)
---------------------
無し。何も書かなくていい。

BSのチャンネル定義(channels.yml)
---------------------
総務省PDFを見ながらnameとchannelを書き、A-PABで対応するserviceIdを調べる。
* name: チャンネル名
* type: BS
* channel: 'BS' + 総務省PDFの物理チャンネル番号2桁 + '_' + 同チャンネルの通し番号(0開始)
    * recfsusb2n.confでの指定と同じなら何でも良さそうだが、recpt1がこのルールでつけているらしく合わせるのがお作法っぽい
* serviceId: A-PABサイトの論理チャンネル番号

``` yaml
# 物理チャンネル1chのサンプル
- name: BS朝日
  type: BS
  channel: BS01_0
  serviceId: 151
- name: BS-TBS
  type: BS
  channel: BS01_1
  serviceId: 161
- name: BSテレ東
  type: BS
  channel: BS01_2
  serviceId: 171
```

BSのチャンネル定義(recfsusb2n.conf)
---------------------
Ch, Freq, SID, TSIDの順で指定する。区切り文字はtab
* Ch: channels.ymlで指定したchannelと合わせる。同じルールで書けばいい
* Freq: 総務省PDFの物理チャンネル番号に100を足したもの
* SID: A-PABサイトの論理チャンネル番号
* TSID: ARIBの資料を元に頑張って作り16進数で書く

``` ini
; 物理チャンネル1chのサンプル
;Ch	Freq	SID	TSID
BS01_0	101	151	0x4010	; BS朝日
BS01_1	101	161	0x4011	; BS-TBS
BS01_2	101	171	0x4012	; BSテレ東
```

TSIDの計算時に必要な放送事業者の追加時期
---------------------
要出典。試しにTSID計算してみてネット上の記事と一致するかどうかで推測したもの。
* 000: 2008年以前に放送を開始したTS
    * 下記以外
* 001: 2010年放送開始のTS
    * 該当なし
* 010: 2011年に既存放送事業者が拡張するTS
    * WOWOWライブ
    * WOWOWシネマ
* 011: 2011年に新規放送事業者が放送開始するTS
    * BSスカパー！
    * 放送大学ex
    * 放送大学on
    * グリーンチャンネル
    * BSアニマックス
    * J SPORTS 1
    * J SPORTS 2
    * J SPORTS 3
    * J SPORTS 4
    * BS釣りビジョン
    * シネフィルWOWOW
    * BS日本映画専門チャンネル
    * ディズニーチャンネル

特殊な計算をするTSID
---------------------
2020/10/30現在の話。今後変わるかもしれない。
* 11chの放送大学とBSスカパー！
    * チャンネルの廃止が関連してる？のかPDFとスロット番号の順序が違う
        * 0: 無し
        * 1: BSスカパー！
        * 2: 放送大学onと放送大学ex(2つの放送が同一のTSID)
* 15chのスターチャンネル
    * スターチャンネル2と3は同一のTSIDを使う

おわりに
---------------------
TSIDの計算が訳わからな過ぎた。なんだこれは。
