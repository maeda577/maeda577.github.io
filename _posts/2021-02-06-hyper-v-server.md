---
layout: post
title: Hyper-V Server 2019のファイル共有周りの初期設定
date: 2021-02-06 14:56 +0900
---
Hyper-V Server 2019の初期設定メモです。ファイル共有かけてWindowsServerバックアップ設定します。

2021/05/15 追記：[サーバーマネージャー関連の手順を追加](https://github.com/maeda577/maeda577.github.io/commit/3bf24245941b1fb29877f5561ae18dd71c58c3b4)

## OSインストール
1. 普通にISOからインストール
1. Administratorパスワードを設定
1. sconfigで以下を設定
    * 2) コンピューター名設定
    * 7) リモートデスクトップ有効化
    * 10) 利用統計を「セキュリティ」
    * 6) WindowsUpdateを複数回
    * 14) で終了。戻りたい時は sconfig で戻れる

以降はPowerShellで設定していく。powershellと打てば起動する。

## App Compatibility FOD
* ある程度の管理ツールとエクスプローラが使えるようになるもの
* [Server Core App Compatibility Feature on Demand (FOD) \| Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/get-started-19/install-fod-19)

``` powershell
# まずは名前調べる
Get-WindowsCapability -Online -Name ServerCore.AppCompatibility*

# 2021/01/31時点ではServerCore.AppCompatibility~~~~0.0.1.0だった
Add-WindowsCapability -Online -Name ServerCore.AppCompatibility~~~~0.0.1.0

# RestartNeeded : True と出たので再起動しておく
Restart-Computer
```

## ファイル共有

``` powershell
# フォルダ作って共有する
New-Item -Path C:/Share -ItemType Directory
New-SmbShare -Path C:/Share -Name share
Grant-SmbShareAccess -Name share -AccountName Administrators -AccessRight Full

# 普通にエクスプローラ上げて共有作ってもいい
explorer.exe

# リモートからサーバーマネージャー経由で共有を作る場合は
# File Serverの機能を入れておく
Get-WindowsFeature
Install-WindowsFeature -Name FS-FileServer
```

## Windows Server バックアップ
* 機能自体はすんなり入るが、GUIが無いのでPowerShellで頑張る
    * [WindowsServerBackup Module \| Microsoft Docs](https://docs.microsoft.com/en-us/powershell/module/windowserverbackup/?view=winserver2012r2-ps)
* バックアップ条件は以下
    * 全ボリュームのバックアップをディスク2に取る
    * ディスク2はパーティションを全部消しておく
    * バックアップスケジュールは毎日午前5時

``` powershell
# 入れられる機能を見つつWindowsServerバックアップ入れる
Get-WindowsFeature
Install-WindowsFeature -Name Windows-Server-Backup

# 「ディスクの管理」を起動してパーティション構成をよしなに作る
diskmgmt.msc

# とりあえずディスク構成等が得られるコマンドを投げて状況確認する
Get-WBDisk
Get-WBVolume -AllVolumes
Get-WBVirtualMachine

# ポリシー作成(既にある場合は持ってくる)
$policy = New-WBPolicy
#$policy = Get-WBPolicy -Editable

# ベアメタルリカバリとシステム状態をバックアップするよう設定する
Add-WBBareMetalRecovery -Policy $policy
Add-WBSystemState -Policy $policy
# ログの切り詰めなどを行うVssFullBackupに設定する(効果は不明)
Set-WBVssBackupOption -Policy $policy -VssFullBackup

# ディスク一覧を変数に入れておく
$disks = Get-WBDisk
# ディスク2番をバックアップ先にする
$target = New-WBBackupTarget -Disk $disks[2]
Add-WBBackupTarget -Policy $policy -Target $target
# 全ボリュームをバックアップ対象にする(NTFS/ReFS以外はバックアップ取れない警告が出るが気にしない)
$volumes = Get-WBVolume -AllVolumes
Add-WBVolume -Policy $policy -Volume $volumes

# 午前5時にバックアップ取るようにする
Set-WBSchedule -Policy $policy -Schedule 05:00

# バックアップ設定を確定
Set-WBPolicy -Policy $policy -AllowDeleteOldBackups
# 確定できたか見る
Get-WBPolicy

# バックアップを即時実行
Get-WBPolicy | Start-WBBackup
# うまくいったかを見る
Get-WBBackupSet

# 次のバックアップ日時
Get-WBSummary
```

## サーバーマネージャーを使ってリモート接続
* 名前解決エラーが出るときはクライアント側でhostsファイルに書く
* FQDNで指定する時はホスト名でも解決できないとエラーが出る
    * hyperv01.hogehuga.com だけでなくhyperv01でも引ける、みたいなイメージ
    * この場合はhostsじゃなくてプライマリDNSサフィックスを設定した方がよさそう
* ググるとCredSSPを有効化する記事が多いが、現在のデフォルト値はドメイン環境でKerberos、ワークグループ環境でNTLMっぽいので無理に有効化する必要もなさそう
    * [WinRM を使用した PowerShell リモート処理のセキュリティに関する考慮事項 - PowerShell \| Microsoft Docs](https://docs.microsoft.com/ja-jp/powershell/scripting/learn/remoting/winrmsecurity?view=powershell-7.1)

``` powershell
# ネットワークプロファイルがパブリックの場合、WinRM用のTCP/5985は同一セグメントからしか繋がらない
Get-NetConnectionProfile
Get-NetFirewallRule -Name WINRM-HTTP-In-TCP-PUBLIC
Get-NetFirewallRule -Name WINRM-HTTP-In-TCP-PUBLIC | Get-NetFirewallAddressFilter
# 必要なセグメントからも許可する
Set-NetFirewallRule -Name WINRM-HTTP-In-TCP-PUBLIC -RemoteAddress LocalSubnet,172.16.0.0/255.240.0.0,192.168.0.0/255.255.0.0
# GUIから設定してもいい。ルールの日本語名は「Windows リモート管理 (HTTP 受信)」
wf.msc

# ネットワークプロファイルを変えてしまう場合は以下
# Get-NetConnectionProfile -InterfaceIndex 7 | Set-NetConnectionProfile -NetworkCategory Private

# リモート管理の有効化
# Set-WSManQuickConfig で一気にやってくれるらしいが、Hyper-V Serverだとデフォルトで有効化されいるっぽい

# WinRMサービスが起動していることを確認する
Get-Service -Name WinRM
# HTTP・5985ポートでListenしているのを確認
Get-WSManInstance -ResourceURI winrm/config/Listener -Enumerate

# CredSSP認証の有効化(これはデフォルトでは有効じゃなかった。無くても動きそう)
# Get-WSManCredSSP
# Enable-WSManCredSSP -Role Server
```

* 以下は接続するクライアント側で管理者権限PowerShellを起動して実行

``` powershell
# オプション機能の一覧を取得
Get-WindowsCapability -Online
# サーバーマネージャーをインストール
Add-WindowsCapability -Online -Name Rsat.ServerManager.Tools~~~~0.0.1.0

# リモート管理での認証の有効化
Set-Item wsman:\localhost\Client\TrustedHosts *
# 厳密に設定するならホスト名をカンマ区切りで入れる
# Set-Item wsman:\localhost\Client\TrustedHosts hv01.contoso.com
```

* あとはサーバーマネージャーやHyper-Vマネージャーなどでつなぐ
    * 認証情報のIDは`接続先ホスト名\Administrator`の形式で入れると良さそう

## (失敗)Windowsコンテナの有効化
* 折角なのでコンテナも使いたかったけどライセンス的に無理そう
    * [Microsoft Software EULA Windows Containers \| Microsoft Docs](https://docs.microsoft.com/ja-jp/virtualization/windowscontainers/images-eula)
    * Hyper-V Serverに対して何も言及されていない…

> The Container Image may only be used with a validly licensed copy of:
>
> Windows Server Standard or Windows Server Datacenter software (collectively “Server Host Software”), or Microsoft Windows Operating System (version 10) software (“Client Host Software”), or Windows 10 IoT Enterprise and Windows 10 IoT Core (collectively “IoT Host Software”). The Server Host Software, Client Host Software, and IoT Host Software are collectively referred to as the “Host Software” and a license for Host Software is a "Host License".

* 手順的にはここらへん
    * Microsoft版手順 [Windows オペレーティング システム コンテナーの準備 \| Microsoft Docs](https://docs.microsoft.com/ja-jp/virtualization/windowscontainers/quick-start/set-up-environment?tabs=Windows-Server)
    * Mirantis版手順 [Install MCR on Windows Servers - MCR documentation](https://docs.mirantis.com/mcr/20.10/install/mcr-windows.html)
* Docker Enterprise事業はMirantisに買収されたらしいので、Mirantis版の方が新しそう。ただライセンスがやっぱり分からない…
    * [山市良のえぬなんとかわーるど: ［速報］Docker Enterprise 改め、Mirantis Container Runtime (MCR) リリース（Engine 19.03.13）](https://yamanxworld.blogspot.com/2020/11/docker-enterprise-mirantis-container.html)
* 実際にやってみたら `Enable-WindowsOptionalFeature -Online -FeatureName containers` で失敗したのでそもそも無理っぽい
