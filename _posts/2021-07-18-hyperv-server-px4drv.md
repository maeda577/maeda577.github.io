---
layout: post
title: Hyper-V Server 2019でPX-W3PE5とmirakurunを動かす
date: 2021-07-18 10:48 +0900
---
* PLEX系チューナーのドライバである[nns779/px4_drv](https://github.com/nns779/px4_drv)がWindows対応した(最高すぎる)のでHyper-V Serverで動かしたメモです
* 公式ドライバはBDAドライバに依存するのでWindowsServerでは動きませんが、px4_drv版はWinUSBで依存なく直接操作するので問題なく動きました(最高)

前提条件
-------------------
* チューナーはPX-W3PE5 1枚のみ
* OSはHyper-V Server 2019
* 物理サーバはHPE Proliant Microserver Gen10 Plus
* ICカードリーダーはROCKETEK RT-SCR3
    * 中身はAlcor Micro AU9540らしい

下準備
-------------------
* 以降のコマンドはすべてPowerShellで実行する
* コマンドプロンプトのウィンドウで`start powershell`すればPowerShellウィンドウが開く

``` powershell
# エクスプローラー等をインストールして再起動
Add-WindowsCapability -Online -Name ServerCore.AppCompatibility~~~~0.0.1.0
Restart-Computer

# 作業ディレクトリの作成
mkdir C:\DTV
mkdir C:\DTV\bin
mkdir C:\DTV\work
mkdir C:\DTV\work\inf

# 以降の作業は全てworkで行う
cd C:\DTV\work

# px4_drvのdevelopブランチをダウンロードしてくる
# 以前はwinusbブランチだったがマージされた
Invoke-WebRequest https://github.com/nns779/px4_drv/archive/refs/heads/develop.zip -OutFile px4_drv.zip
Expand-Archive -Path .\px4_drv.zip -DestinationPath .\
```

ドライバの署名
-------------------
* 起動時にF8連打して「ドライバー署名の強制を無効にする」を使う場合は不要

``` powershell
# Windows Driver Kitのダウンロードとインストール(Inf2Cat.exeを使うためだけに入れる)
Invoke-WebRequest https://go.microsoft.com/fwlink/?linkid=2166289 -OutFile wdksetup.exe
.\wdksetup.exe

# infにCatalogFileの定義を加えて別ディレクトリに入れる
# ディレクトリ分けるのはInf2Catの引数でディレクトリを指定する必要があるため
Get-Content .\px4_drv-develop\winusb\pkg\inf\pxw3pe5_winusb.inf |
  ForEach-Object { $_.Replace("[Version]","[Version]`r`nCatalogFile.NTAMD64 = pxw3pe5_winusb.cat") } |
  Set-Content .\inf\pxw3pe5_winusb.inf

# カタログファイルを作る
& 'C:\Program Files (x86)\Windows Kits\10\bin\10.0.22000.0\x86\Inf2Cat.exe' /os:ServerRS5_X64,Server2016_X64 /driver:.\inf\
# .catができてるはず
dir .\inf

# 自己署名証明書を作ってTrustedPublisherに入れる
New-SelfSignedCertificate -Subject "CN=px4SelfSigned" -TestRoot -Type CodeSigningCert -CertStoreLocation Cert:\LocalMachine\My\ -NotAfter (Get-Date).AddYears(10) |
  Move-Item -Destination Cert:\LocalMachine\TrustedPublisher\
# 証明書はGet-ChildItemで見れる。-TestRootを付けたのでIssuerはCertReq Test Rootになっているはず
Get-ChildItem -Path Cert:\LocalMachine\TrustedPublisher\ | Format-List
# CertReq Test Rootを信頼されたルート証明機関にする
Get-ChildItem -Path Cert:\LocalMachine\CA\ -DnsName "CertReq Test Root" |
  Move-Item -Destination Cert:\LocalMachine\Root\ 

# ドライバを署名する(署名するのはcatの方のみ)
$cert = Get-ChildItem -Path Cert:\LocalMachine\TrustedPublisher\ -CodeSigningCert -DnsName "px4SelfSigned"
Set-AuthenticodeSignature -FilePath .\inf\pxw3pe5_winusb.cat -Certificate $cert -TimestampServer http://timestamp.digicert.com/?alg=sha1
```

ドライバのインストール
-------------------
* devcon.exeがない場合は、`devmgmt`でデバイスマネージャーを起動して右クリックから`ハードウェア変更のスキャン`
    * `pnputil /scan-devices`とかで出来るらしいがHyper-V Serverでは未実装のオプションだった
* エラーで`レジストリ内の構成情報が不完全であるか、または壊れているためこのハードウェア デバイスを開始できません。`となった場合は、以下を見つつレジストリの`UpperFilters`のエントリを消すとうまくいくかもしれない
    * [CD ドライブまたは DVD ドライブが Windows やその他のプログラムにより認識されない](https://support.microsoft.com/ja-jp/topic/cd-%E3%83%89%E3%83%A9%E3%82%A4%E3%83%96%E3%81%BE%E3%81%9F%E3%81%AF-dvd-%E3%83%89%E3%83%A9%E3%82%A4%E3%83%96%E3%81%8C-windows-%E3%82%84%E3%81%9D%E3%81%AE%E4%BB%96%E3%81%AE%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%A0%E3%81%AB%E3%82%88%E3%82%8A%E8%AA%8D%E8%AD%98%E3%81%95%E3%82%8C%E3%81%AA%E3%81%84-64da3690-4c1d-ef04-63b8-cf9cc38ca53e)

``` powershell
# 「PXW3PE5」が認識されていることを確認
Get-PnpDevice -FriendlyName *W3PE*
# ドライバインストール
pnputil /add-driver .\inf\pxw3pe5_winusb.inf /install
# 名前が「PLEX PX-W3PE5 ISDB-T/S Receiver Device (WinUSB)」に変わっているはず
Get-PnpDevice -FriendlyName *W3PE*
# StatusがOKになっていない場合、何かおかしいのでエラー内容を見つつデバイススキャンや上記URLのレジストリ操作を試す
Get-PnpDevice -FriendlyName *W3PE* | Format-List

# デバイスがいない場合はデバイスマネージャーの「ハードウェア変更のスキャン」相当をしてみる
& "C:\Program Files (x86)\Windows Kits\10\Tools\x64\devcon.exe" rescan
```

px4_drvのWinUSB版ビルド
-------------------

* Build Tools for Visual Studio 2019をダウンロードする公式URLは以下
    * [Visual Studio をダウンロードいただきありがとうございます - Visual Studio](https://visualstudio.microsoft.com/ja/thank-you-downloading-visual-studio/?sku=BuildTools&rel=16)
    * これをPowerShellでDLしようとするとただのHTMLが返ってくる
* Dockerに入れる手順にDL用URLが書かれているのでそっちを使う
    * [Visual Studio Build Tools をコンテナーにインストールする \| Microsoft Docs](https://docs.microsoft.com/ja-jp/visualstudio/install/build-tools-container?view=vs-2019)
* 他のマシンでビルドする場合も[Visual C++ Redistributable x64](https://aka.ms/vs/16/release/VC_redist.x64.exe)だけは入れる

``` powershell
# Build Tools for Visual Studio 2019を入れる
# ワークロードは「C++によるデスクトップ開発」をチェックする。右の方に「MSVC v142」のようにバージョンが書かれているので一応確認する
Invoke-WebRequest https://aka.ms/vs/16/release/vs_buildtools.exe -OutFile vs_buildtools.exe
.\vs_buildtools.exe

# Visual Studio 2019 Developer PowerShellに切り替える
& "C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildTools\Common7\Tools\Launch-VsDevShell.ps1"
# ビルド
MSBuild .\px4_drv-develop\winusb\px4_winusb.sln /t:clean /t:build /p:Configuration=Release /p:Platform=x64

# ビルドしたものを移動しておく
Copy-Item .\px4_drv-develop\winusb\build\x64\Release\BonDriver_PX4.dll ..\bin\BonDriver_PX4-T.dll
Copy-Item .\px4_drv-develop\winusb\build\x64\Release\BonDriver_PX4.dll ..\bin\BonDriver_PX4-S.dll
Copy-Item .\px4_drv-develop\winusb\build\x64\Release\DriverHost_PX4.exe ..\bin\

# 設定ファイルのコピー
Copy-Item .\px4_drv-develop\winusb\pkg\BonDriver_PX4\BonDriver_PX4-S.ChSet.txt ..\bin\
Copy-Item .\px4_drv-develop\winusb\pkg\BonDriver_PX4\BonDriver_PX4-T.ChSet.txt ..\bin\
Copy-Item .\px4_drv-develop\winusb\pkg\BonDriver_PX4\BonDriver_PX4-S.ini ..\bin\
Copy-Item .\px4_drv-develop\winusb\pkg\BonDriver_PX4\BonDriver_PX4-T.ini ..\bin\
Copy-Item .\px4_drv-develop\winusb\pkg\DriverHost_PX4\DriverHost_PX4.ini ..\bin\

# firmware抽出用のファイルもコピー
Copy-Item .\px4_drv-develop\winusb\build\x64\Release\fwtool.exe .\
Copy-Item .\px4_drv-develop\fwtool\fwinfo.tsv .\
```

firmwareの抽出
-------------------

* 抽出するfirmwareはどれでもいいが、0b41a994のものがいいらしい
    * [nns779さんはTwitterを使っています 「WinUSB版でも it930x-firmware.bin はCRC32が 0b41a994 のものがおすすめです」 / Twitter](https://twitter.com/yamask32/status/1413504578411134988)
    * [Windows10 で EPGStation セットアップ(2021年7月) - Qiita](https://qiita.com/Daigorian/items/4895233acc893955b45d) より引用

``` powershell
# 公式ドライバをダウンロードして展開
Invoke-WebRequest -Uri http://plex-net.co.jp/download/pxq3u4v1.2.zip -OutFile pxq3u4v1.2.zip
Expand-Archive -Path .\pxq3u4v1.2.zip
# firmware抽出
.\fwtool.exe .\pxq3u4v1.2\x64\PXQ3U4.sys ..\bin\it930x-firmware.bin
```

libaribb25のビルド
-------------------
``` powershell
# ソースコードのダウンロードと展開
Invoke-WebRequest https://github.com/epgdatacapbon/libaribb25/archive/refs/heads/master.zip -OutFile libaribb25.zip
Expand-Archive -Path .\libaribb25.zip -DestinationPath .\

# VisualStudioの[ソリューションの再ターゲット]相当の作業
# PlatformToolsetの値はBuild Toolsを入れた際に確認したMSVCのバージョン
Get-ChildItem -Path .\ -Filter *.vcxproj -Recurse | 
    ForEach-Object {
        $fullName = $_.FullName;
        (Get-Content $_.FullName) | 
        ForEach-Object { $_ -replace "<WindowsTargetPlatformVersion>.+</WindowsTargetPlatformVersion>", "<WindowsTargetPlatformVersion>10.0</WindowsTargetPlatformVersion>" } |
        ForEach-Object { $_ -replace "<PlatformToolset>.+</PlatformToolset>", "<PlatformToolset>v142</PlatformToolset>" } |    
        Set-Content $fullName;
    }

# ビルド
MSBuild .\libaribb25-master\arib_std_b25.sln /t:clean /t:build /p:Configuration=Release /p:Platform=x64

# リネームしつつ移動
Copy-Item .\libaribb25-master\x64\Release\libaribb25.dll ..\bin\B25Decoder.dll
```
* 以降はDeveloper PowerShellではなく普通のPowerShellでもいい

BonRecTestのダウンロードとテスト実行
-------------------
* [rndomhack/BonRecTest: Recoding test program for BonDriver](https://github.com/rndomhack/BonRecTest)
* Windows10に比べ標準ドライバの数が少ない？のかICカードリーダーのドライバは手で入れておく必要があった
* RDP経由ではICカードが読めないため直接操作する

``` powershell
# ダウンロードして展開
Invoke-WebRequest https://github.com/rndomhack/BonRecTest/releases/download/v1.0.0/BonRecTest_v1.0.0.zip -OutFile BonRecTest_v1.0.0.zip
Expand-Archive -Path .\BonRecTest_v1.0.0.zip
# 移動
Copy-Item .\BonRecTest_v1.0.0\x64\BonRecTest.exe ..\bin\

# テスト実行。10秒ぐらい待って何も出なければCtrl+cで止める
# output.tsのファイルサイズが膨らんでいればOK
# CreateFileW() failed.が出た場合、無視して進めてEPGStation経由なら動くかもしれない
..\bin\BonRecTest.exe --driver BonDriver_PX4-T.dll --output output.ts --channel 14 --decoder B25Decoder.dll
```

Node.jsとmirakurunのインストール
-------------------
* Node.jsのURLは適宜変える
* mirakurunの設定はググれば出てくる
    * channels.ymlの地上波のチャンネル番号はLinux版と13ずれる

``` powershell
# Node.jsインストール。設定は全部デフォルトで入れる
Invoke-WebRequest https://nodejs.org/dist/v14.17.3/node-v14.17.3-x64.msi -OutFile node-v14.17.3-x64.msi
./node-v14.17.3-x64.msi
# インストール先にPATH通したのを反映するため一旦logoffしてもう一回入る
logoff
# カレントを戻す
start powershell
cd C:\DTV\work

# Mirakurunインストール
npm install --global winser@1.0.3
npm install --global --production mirakurun@latest
# バージョン情報が返ってくるか試す
Invoke-RestMethod http://localhost:40772/api/status
# configはユーザープロファイルに入っているのでよしなに設定する
# tunersは下に書いたサンプル通りでいいはず
notepad C:\Users\Administrator\.Mirakurun\tuners.yml
# channelsは地域に合わせて
notepad C:\Users\Administrator\.Mirakurun\channels.yml
# serverはそのままでいい
notepad C:\Users\Administrator\.Mirakurun\server.yml
# 設定したらmirakurun再起動
Restart-Service mirakurun

# Windowsのファイアウォールで40772を許可する
New-NetFirewallRule -Name Mirakurun-In-TCP -DisplayName "Mirakurun (TCP 受信)" -Direction Inbound -LocalPort 40772 -Protocol TCP -RemoteAddress 192.168.0.0/16 -Program %PROGRAMFILES%\nodejs\node.exe -Action Allow
```

* tuners.ymlはこんな感じ

``` yaml
- name: PX4-T1
  types:
    - GR
  command: C:\DTV\bin\BonRecTest.exe --driver BonDriver_PX4-T.dll --decoder B25Decoder.dll --output - --space <space> --channel <channel>
- name: PX4-T2
  types:
    - GR
  command: C:\DTV\bin\BonRecTest.exe --driver BonDriver_PX4-T.dll --decoder B25Decoder.dll --output - --space <space> --channel <channel>
- name: PX4-S1
  types:
    - BS
    - CS
  command: C:\DTV\bin\BonRecTest.exe --driver BonDriver_PX4-S.dll --decoder B25Decoder.dll --output - --space <space> --channel <channel>
- name: PX4-S2
  types:
    - BS
    - CS
  command: C:\DTV\bin\BonRecTest.exe --driver BonDriver_PX4-S.dll --decoder B25Decoder.dll --output - --space <space> --channel <channel>
```

* 他のPCから `http://<ip_address>:40772/` を開きMirakurunのWebUIが開けばOK
* EPGStaionやChinachuなどは適当にHyper-V上の仮想マシンで構築する

起動時にデバイススキャンする
-------------------
* 再起動するとPX-W3PE5を見失うことが多かった
    * タスクスケジューラで起動時にデバイススキャンをかけるようにしたら安定した
* devcon.exeが必要。いちいちWDKを入れたくない場合はdevconだけ取り出せるらしい
    * [Windows Driver Kitをインストールすることなしにdevcon.exeを用意する – guro_chanの手帳](https://www7390uo.sakura.ne.jp/wordpress/archives/705)

``` powershell
# 起動時にタスクを起動するトリガー
$trigger = New-ScheduledTaskTrigger -AtStartUp
# devconを起動するアクション
$action = New-ScheduledTaskAction -Execute "C:\Program Files (x86)\Windows Kits\10\Tools\x64\devcon.exe" -Argument "rescan"
# Administratorで起動する
$principal = New-ScheduledTaskPrincipal -UserId SYSTEM -LogonType ServiceAccount
# タスク登録
Register-ScheduledTask -TaskName "RescanDevice" -Trigger $trigger -Action $action -Principal $principal
```

再起動して不調になった時にやることメモ
-------------------
* まずはテスト実行
    * `C:\DTV\bin\BonRecTest.exe --driver BonDriver_PX4-T.dll --output C:\DTV\bin\output.ts --channel 14 --decoder B25Decoder.dll`
* チューナーが見つからない、的なエラー
    * `devmgmt`で`ハードウェア変更のスキャン`
* `px4::DeviceBase::DeviceBase: CreateFileW() failed.` と `BonDriver::OpenTuner: WaitForSingleObject() failed.`
    * 原因わからず、しばらく待ってみたら(10分以上)解消したこともある
    * この状態でもmirakurunのEPG情報は取れていたり、EPGStation経由で見れたりする
    * 2021/09/01時点のdevelopブランチの最新版をビルドして差し替えたら直った感じがする
* mirakurunのWebUIが開かない
    * `Restart-Service mirakurun`
