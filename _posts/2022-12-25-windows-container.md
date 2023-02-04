---
layout: post
title: Windows Serverにdockerをインストールする
date: 2022-12-25 21:03 +0900
---
ヤフオクで買ったサーバにWindows Server 2016のCOAラベルがついていたので、OS入れてdockerを入れました。

## はじめに

* 以下ドキュメントを読むのが正確
    * [Windows のコンテナーに関するドキュメント \| Microsoft Learn](https://learn.microsoft.com/ja-jp/virtualization/windowscontainers/)
    * なお読んでいない
* 検証に使用したOSはWindows Server 2016 Standard

## ランタイムの選定

* 以下によると、現在は3択
    * [Windows オペレーティング システム コンテナーの準備 \| Microsoft Learn](https://learn.microsoft.com/ja-jp/virtualization/windowscontainers/quick-start/set-up-environment?tabs=containerd)
        * > Windows で現在サポートされているランタイムは、containerd、Moby、Mirantis Container Runtime です。
* Mobyとcontainerdにはインストールスクリプトも用意されている

1. containerd
    * 公式: [containerd – An industry-standard container runtime with an emphasis on simplicity, robustness and portability](https://containerd.io/)
    * Kubernetesでもお馴染みのcontainerd。実はWindowsでも動く。[リリースページにはWindows向けビルドもある](https://github.com/containerd/containerd/releases)
    * インストールスクリプトは[install-containerd-runtime.ps1](https://raw.githubusercontent.com/microsoft/Windows-Containers/Main/helpful_tools/Install-ContainerdRuntime/install-containerd-runtime.ps1)
    * **やってみたがコンテナイメージが壊れたりWindowsDefenderが壊れたりしたので使うのやめた**
1. Moby
    * 公式: [Moby](https://mobyproject.org/)
    * だいたいDocker Community Editionのこと
    * インストールスクリプトは[install-docker-ce.ps1](https://raw.githubusercontent.com/microsoft/Windows-Containers/Main/helpful_tools/Install-DockerCE/install-docker-ce.ps1)
    * インストールスクリプトを読むと [https://master.dockerproject.org/](https://master.dockerproject.org/) からdocker.exeとdockerd.exeをダウンロードし、それをSystem32に入れている。System32はちょっと気持ち悪い
    * Docker公式の手順だとProgramFilesに入れるのでしっくりくる。こっちに沿って作業し、ついでにcomposeも入れる。
        * [Install Docker Engine from binaries \| Docker Documentation](https://docs.docker.com/engine/install/binaries/#install-server-and-client-binaries-on-windows)
        * [Install the Compose standalone \| Docker Documentation](https://docs.docker.com/compose/install/other/)
1. Mirantis Container Runtime
    * 公式: [Container Runtime Interface Tools \| Swarm & Kubernetes Containers](https://www.mirantis.com/software/container-runtime/)
    * かつてDocker Enterprise Editionと呼ばれていたもの
        * [「Docker Enterprise」をMirantisが買収　Dockerは開発者ビジネスに専念 - ITmedia NEWS](https://www.itmedia.co.jp/news/articles/1911/14/news078.html)
    * 有償。たぶん商用利用でサポートが欲しい人向け。個人利用で選ぶ理由は思いつかなかった

今回はMobyを入れる

## Windowsコンテナ機能の準備

* Windowsコンテナにはプロセス分離とHyper-V分離がある
    * [分離モード \| Microsoft Learn](https://learn.microsoft.com/ja-jp/virtualization/windowscontainers/manage-containers/hyperv-container)
    * プロセス分離
        * Linuxにおけるコンテナと同じ感じ。Windowsの中でWindowsコンテナが動く
        * カーネルはホスト側と共有する
        * WindowsServerだとデフォルト値はこっち
    * Hyper-V分離
        * Docker Desktopみたいな感じ。Hyper-Vの仮想マシンが作られて、その中でコンテナが動く
        * カーネルはホスト側と分離される
        * Windows上でLinuxコンテナを動かしたい場合はこっち
        * windows10にDocker Desktop入れた時はこっち
* Docker Engineではプロセス分離しか動かない
    * [Install Docker Engine from binaries \| Docker Documentation](https://docs.docker.com/engine/install/binaries/#install-server-and-client-binaries-on-windows)
        * > On Windows, these binaries only provide the ability to run native Windows containers (not Linux containers).

``` powershell
# コンテナ機能の有効化
Install-Feature -FeatureName Containers
# プロセス分離だけでなくHyper-V分離も使いたい場合はHyper-Vも入れる。dockerはHyper-V分離対応していないので入れていない
#Install-Feature -FeatureName Hyper-V
# 再起動
Restart-Computer

# コンテナ用ネットワークの確認。今は何も無いはず
Get-ContainerNetwork
```

* ここで追加されたのはHCS/HNSとそのAPI
    * HCS/HNSは以下ページの図が分かりやすい
    * [Windows コンテナー ネットワーク \| Microsoft Learn](https://learn.microsoft.com/ja-jp/virtualization/windowscontainers/container-networking/architecture)
    * 管理ツールは入らないので、この段階ではコンテナ作成できない
    * New-Containerなどのコマンドがあったらしいが、既に開発中止になった模様
        * [PowerShellでDockerを操作する方法についてあれやこれや - しばたテックブログ](https://blog.shibata.tech/entry/2016/10/21/000157)
        * [microsoft/Docker-PowerShell: PowerShell Module for Docker](https://github.com/Microsoft/Docker-PowerShell/)
* HNSの管理PowerShellコマンドは入る
    * Get-ContainerNetwork など。`Get-Command -Module Containers` で見ると3つ
    * HNSというPowerShellモジュールを入れるのが近代的っぽい？が特に必要性を感じなかった
        * [PowerShell Gallery \| HNS 0.2.4](https://www.powershellgallery.com/packages/HNS/0.2.4)
* hcsdiag.exeというのも追加されるらしいがWindowsServer2016では入らなかった。謎

## Dockerインストール

``` powershell
# インストールするDockerのバージョン番号。リリースページを見て適宜変更 https://download.docker.com/win/static/stable/x86_64/
$DockerVersion = "20.10.21"

# dockerバイナリのダウンロード
Invoke-WebRequest -Uri "https://download.docker.com/win/static/stable/x86_64/docker-$DockerVersion.zip" -OutFile "~/Downloads/docker-$DockerVersion.zip" -UseBasicParsing
# 展開。展開先はProgramFiles配下
Expand-Archive -Path "~/Downloads/docker-$DockerVersion.zip" -DestinationPath $Env:ProgramFiles

# Windowsサービスに登録
& "$Env:ProgramFiles/docker/dockerd.exe" --register-service
# サービス開始。自動起動設定になっているので次回からは不要
Start-Service -Name docker

# ネットワーク確認。natとnoneが出来てるはず
& "$Env:ProgramFiles/docker/docker.exe" network ls
# 内部的にHNSを叩いているっぽい？のでpowershellでもnatネットワークが見える
Get-ContainerNetwork
```

## Docker Composeインストール

なにかと便利なDocker Composeも入れておく。必須ではない

``` powershell
# インストールするDocker Compose v2のバージョン番号。リリースページを見て適宜変更 https://github.com/docker/compose/releases
$DockerComposeVersion = "v2.14.2"

# docker-composeバイナリのダウンロード。直接実行せずブラグインとして使うのでcli-pluginsに入れる
[Net.ServicePointManager]::SecurityProtocol += [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest -Uri "https://github.com/docker/compose/releases/download/$DockerComposeVersion/docker-compose-windows-x86_64.exe" -OutFile "$ENV:ProgramFiles/docker/cli-plugins/docker-compose.exe"
```

## パスを通す

不便なのでパスを通す。必須ではない。

``` powershell
# dockerのパスを通す
$NewPath = [Environment]::GetEnvironmentVariable('PATH', 'Machine')
$NewPath = "$NewPath;$ENV:ProgramFiles\docker"
[Environment]::SetEnvironmentVariable('PATH' , $NewPath , 'Machine')
# パス反映のため一度ログオフ
logoff
# 確認。バージョンが出るはず
dockerd.exe -v
docker.exe version
# composeも読まれているはず
docker.exe compose version
```

## コンテナ起動

ここから非常に遅い

* WindowsServer2016なのでservercoreイメージぐらいしかなく、それを起動すると約6GBのダウンロードが走る。5分ぐらいかかる
* その後、Extractingしているようで、SSDを積んでいてもさらに10分くらいかかる
* 今回はservercoreを起動してみたが、より新しいServerOSを使っているならnanoserverの方がテストに向いているはず
    * [Windows コンテナーの基本イメージ \| Microsoft Learn](https://learn.microsoft.com/ja-jp/virtualization/windowscontainers/manage-containers/container-base-images)

``` powershell
# コンテナイメージをダウンロード
docker.exe image pull mcr.microsoft.com/windows/servercore:ltsc2016
# 起動してみる。pingが届くことを確認。--rmがついているのでコンテナは即削除される
docker.exe run --rm mcr.microsoft.com/windows/servercore:ltsc2016 ping 8.8.8.8
# -itをつけて対話的な操作もできる
docker.exe run -it --rm mcr.microsoft.com/windows/servercore:ltsc2016 powershell
```

* -itで対話操作している時、コピペが50文字で切られる問題が起きる
    * [Paste limited to 50 chars when run in Docker container on Windows · Issue #460 · PowerShell/PSReadLine](https://github.com/PowerShell/PSReadLine/issues/460)
* 原因はPSReadLineなので、コンテナ内で `Remove-Module PSReadLine` すれば50文字以上も貼り付けられる
    * 色分けされず履歴も保持されなくなるので不便。 `Import-Module PSReadLine` すれば戻る

これでdockerインストールは完了

# (失敗) containerdインストール

以降、やってみたけど実用的ではなかった手順。悔しいので残す。

## ネットワーク作成

* コンテナが使う仮想ネットワークインタフェースを作る。ModeはTransparent
    * Transparentは言葉そのままに透過するモード。外部に直結するイメージ。FirewallはホストOS側のが効く。DHCPなんかも外の物を使う
    * WindowsServer2016だと、ModeはTransparent, NAT, L2Bridge, L2Tunnelが選べた。挙動は以下
    * [Windows コンテナーネットワークドライバー \| Microsoft Learn](https://learn.microsoft.com/ja-jp/virtualization/windowscontainers/container-networking/network-drivers-topologies)

``` powershell
# どのネットワークアダプタを使うか決まっている場合はそれを取得
$netAdapter = Get-NetAdapter -Name LAN1
# 決まっていなければUPしているやつを取得
#$netAdapter = (Get-NetAdapter | Where-Object {($_.Status -eq 'Up') -and ($_.ConnectorPresent)})[0]
# Transparentネットワークを作る。なんでTransparentなのかは分からないが、スクリプトがそうなっているので従う
New-ContainerNetwork -Name "Transparent" -Mode Transparent -NetworkAdapterName $netAdapter.Name
```

* 「ネットワークと共有センター」のNIC一覧とか見ると増えたのが分かる

## 下準備

* Invoke-WebRequestが失敗するなーと思ったらTLS1.2を有効化する。多分powershellプロセス単位
    * [Powershell：Invoke-WebRequestに失敗する場合の対処方法（TLS1.2有効化）](https://zenn.dev/hotaka_noda/articles/37e6a0f178039a)

``` powershell
# TLS1.2の一時有効化
[Net.ServicePointManager]::SecurityProtocol += [Net.SecurityProtocolType]::Tls12
```

## tarインストール

* 後の作業でtar.gzを解凍するため
* Windows Server 2016ではインストールが必要。恐らく新しめのWindowsServerやWindows11などでは最初から入っている
    * [Tar and Curl Come to Windows! \| Microsoft Learn](https://learn.microsoft.com/ja-jp/virtualization/community/team-blog/2017/20171219-tar-and-curl-come-to-windows)
* 上記ブログで、libarchiveのtarを使っていると書いてあるので合わせる
    * [libarchive - C library and command-line tools for reading and writing tar, cpio, zip, ISO, and other archive formats @ GitHub](http://libarchive.org/)
* libarchiveはVisual C++ 再頒布可能パッケージが必要だった。ダウンロードURLは以下に書いてある
    * [サポートされている最新の Visual C++ 再頒布可能パッケージのダウンロード \| Microsoft Learn](https://learn.microsoft.com/ja-jp/cpp/windows/latest-supported-vc-redist?view=msvc-170)
    * VCRUNTIME140.dll を要求されたが、バージョンは気にせず最新のVisualC++を入れていいらしい

``` powershell
# Visual C++パッケージダウンロードとインストール
Invoke-WebRequest -Uri https://aka.ms/vs/17/release/vc_redist.x64.exe -OutFile "~/Downloads/VC_redist.x64.exe" -UseBasicParsing
& "~/Downloads/VC_redist.x64.exe"

# インストールするlibarchiveのバージョン番号。リリースページを見て適宜変更 https://github.com/libarchive/libarchive/releases
$LibarchveVersion = "3.6.1"
# tarのダウンロードと展開。展開先はProgramFilesの配下
Invoke-WebRequest -Uri "https://github.com/libarchive/libarchive/releases/download/v$LibarchveVersion/libarchive-v$LibarchveVersion-amd64.zip" -OutFile "~/Downloads/libarchive-v$LibarchveVersion-amd64.zip" -UseBasicParsing
Expand-Archive -Path "~/Downloads/libarchive-v$LibarchveVersion-amd64.zip" -DestinationPath $ENV:ProgramFiles
```

## containerdインストール

``` powershell
# インストールするcontainerdのバージョン番号。リリースページを見て適宜変更 https://github.com/containerd/containerd/releases
$ContainerDVersion = "1.6.10"
# ダウンロード
Invoke-WebRequest -Uri "https://github.com/containerd/containerd/releases/download/v$ContainerDVersion/containerd-$ContainerDVersion-windows-amd64.tar.gz" -OutFile "~/Downloads/containerd-$ContainerDVersion-windows-amd64.tar.gz" -UseBasicParsing
# 展開。展開先はProgramFiles配下。これはcontainerdのデフォルトconfigに書かれているので、変えない方がいい
New-Item "$ENV:ProgramFiles/containerd" -ItemType Directory
# 新し目のWindowsなら普通にtar.exeでいい
& "$ENV:ProgramFiles/libarchive/bin/bsdtar.exe" -xvf "$HOME/Downloads/containerd-$ContainerDVersion-windows-amd64.tar.gz" -C "$ENV:ProgramFiles/containerd"
```

## CNIプラグインのインストール

* containerdからHNSを制御するためのCNIプラグインを入れる（よく分かってない）
    * [microsoft/windows-container-networking: Container networking plugins for Windows containers](https://github.com/microsoft/windows-container-networking)

``` powershell
# インストールするCNIのバージョン番号。リリースページを見て適宜変更 https://github.com/microsoft/windows-container-networking/releases
$WinCNIVersion = "0.3.0"
# ダウンロード
Invoke-WebRequest -Uri "https://github.com/microsoft/windows-container-networking/releases/download/v$WinCNIVersion/windows-container-networking-cni-amd64-v$WinCNIVersion.zip" -OutFile "~/Downloads/windows-container-networking-cni-amd64-v$WinCNIVersion.zip" -UseBasicParsing
# 展開。展開先はcontainerd/cni/bin配下。これもconfigに書かれていたはず
Expand-Archive -Path "~/Downloads/windows-container-networking-cni-amd64-v$WinCNIVersion.zip" -DestinationPath "$ENV:ProgramFiles/containerd/cni/bin"
```

## nerdctlのインストール

* dockerコマンドに相当
    * [containerd/nerdctl: contaiNERD CTL - Docker-compatible CLI for containerd, with support for Compose, Rootless, eStargz, OCIcrypt, IPFS, ...](https://github.com/containerd/nerdctl)
* containerdにはctrコマンドが付属しているが、こっちの方がdockerコマンド互換なので便利らしい
* コマンド体系にこだわりが無ければctrでもいいはず
* 下の方で書くように、ネットワークが動かないのでnerdctlは使わなくなった。なので入れなくてもいい

``` powershell
# インストールするnerdctlのバージョン番号。リリースページを見て適宜変更 https://github.com/containerd/nerdctl/releases
$NerdCTLVersion = "1.0.0"
# ダウンロード
Invoke-WebRequest -Uri "https://github.com/containerd/nerdctl/releases/download/v$NerdCTLVersion/nerdctl-$NerdCTLVersion-windows-amd64.tar.gz" -OutFile "~/Downloads/nerdctl-$NerdCTLVersion-windows-amd64.tar.gz" -UseBasicParsing
# 展開。展開先はProgramFiles配下。どこでもいい気がする
New-Item "$ENV:ProgramFiles/nerdctl" -ItemType Directory
# 新し目のWindowsなら普通にtar.exeでいい
& "$ENV:ProgramFiles/libarchive/bin/bsdtar.exe" -xvf "$HOME/Downloads/nerdctl-$NerdCTLVersion-windows-amd64.tar.gz" -C "$ENV:ProgramFiles/nerdctl"
```

## パスを通す

``` powershell
# containerdとnerdctlのパスを通す
$NewPath = [Environment]::GetEnvironmentVariable('PATH', 'Machine')
$NewPath = "$NewPath;$ENV:ProgramFiles\nerdctl;$ENV:ProgramFiles\containerd\bin"
[Environment]::SetEnvironmentVariable('PATH' , $NewPath , 'Machine')
# パス反映のため一度ログオフ
logoff
# 確認。ヘルプメッセージが出るはず
ctr.exe -h
containerd.exe -h
nerdctl.exe -h
```

## containerd起動

``` powershell
# デフォルトconfigをファイルに出力。この中にファイルパスなどが書かれているため、どうしてもProgramFilesに置きたくない場合はここをいじる
containerd.exe config default | Out-File "$ENV:ProgramFiles/containerd/config.toml" -Encoding ascii
# Windowsサービスに登録
containerd.exe --register-service
# 開始
Start-Service -Name containerd
# 呼んでみる（何もないはず。エラーが出ないことだけ確認）
nerdctl.exe image ls --all
```

**インストールスクリプトはここまで。ここから先は個人的に試したもの**

## コンテナ起動

``` powershell
# コンテナイメージをダウンロード。数GBのダウンロードが走り、さらに展開が行われる。落ちているように見えるが実は処理が進んでいるので、安易にctrl+cしてはいけない
nerdctl.exe pull mcr.microsoft.com/windows/servercore:ltsc2016
# 起動してみる。pingが起動したらOK。pingの送信は失敗するはず
nerdctl.exe run --rm mcr.microsoft.com/windows/servercore:ltsc2016 ping 8.8.8.8
```

ネットワークに繋がらないことに気づく。nerdctlではダメらしい。

* [[Windows] Networking doesn't work · Issue #559 · containerd/nerdctl](https://github.com/containerd/nerdctl/issues/559)
    * `nerdctl network ls` をすると自動でnatネットワークが作られる
    * ipconfigしても何も出ないが、`ctr --cni` とかやると出るらしい
    * `Networking support for Windows is not implemented yet in nerdctl.` とある。ダメそう

さらに、ここで謎の事象が起きる

* なぜかスタートメニューでピン留めしていたサーバーマネージャーのショートカットが消える
    * ServerManager.exe で起動するので気にしない
* スタートメニューの「Windows管理ツール」と中身が英語になる
    * 理由は全く分からないが、使えない訳では無いので気にしない
* 一度サインアウトすると、以降WindowsDefenderが起動しなくなるしコンテナイメージも壊れる
    * `nerdctl image ls` するとイメージが壊れていると表示される
    * さすがにWindowsDefender壊れるのは厳しい

もう何も分からないので諦める

## おわりに

containerd何も分からない
