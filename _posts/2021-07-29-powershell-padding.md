---
layout: post
title: PowerShellでファイル名に含まれる数字を0埋めしてリネームする
date: 2021-07-29 22:24 +0900
---
ファイル名に含まれる数字をいい感じにリネームしたかったので作りました。

やりたいこと
-----------------
ファイル名に含まれる数字を0埋めして、Get-ChildItemした時もいい感じにソートされるようにする。

| 元のファイル名 | 変更後のファイル名 |
| -- | -- |
| hoge1.jpg | hoge001.jpg |
| hoge2.jpg | hoge002.jpg |
| hoge3.jpg | hoge003.jpg |
| hoge11.jpg | hoge011.jpg |

総じて通し番号は最後につける気がするので、最後に出てくる番号だけ0埋めする。

| 元のファイル名 | 変更後のファイル名 |
| -- | -- |
| hoge1huga1.jpg | hoge1huga001.jpg |
| hoge1huga2.jpg | hoge1huga002.jpg |
| hoge1huga3.jpg | hoge1huga003.jpg |
| hoge1huga11.jpg | hoge1huga011.jpg |

作ったもの
-----------------

``` powershell
function Update-FileNameWithNumberPadding {
    [CmdletBinding(SupportsShouldProcess = $true)]
    Param (
        [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
        [ValidateScript( { Test-Path $_.FullName })]
        [System.IO.FileInfo]
        # Target file for update.
        $FileInfo,
        [int]
        # Padding width. Default is 3.
        $TotalWidth = 3,
        [char]
        # Padding character. Default is '0'.
        $PaddingChar = '0'
    )
    Process{
        $selectStr = Select-String -InputObject $FileInfo.Name -Pattern "\d+" -AllMatches
        if ($null -eq $selectStr) { return }
        $lastMatch = $selectStr.Matches | Select-Object -Last 1
        $newName = $FileInfo.Name.Substring(0, $lastMatch.Index) + $lastMatch.Value.PadLeft($TotalWidth, $PaddingChar) + $FileInfo.Name.Substring($lastMatch.Index + $lastMatch.Length)
        Move-Item -Path $FileInfo.FullName -Destination (Join-Path -Path $FileInfo.DirectoryName -ChildPath $newName) -WhatIf:$WhatIfPreference
    }
}
```

使い方
-----------------
1. 上に書いた関数をPowerShellウィンドウにコピペする
1. 対象ファイルがあるディレクトリに移動する
1. 対象ファイルを取得するGet-ChildItem関数を考える
    * `Get-ChildItem -File -Recurse -Include "*.jpg"` など
1. `Get-ChildItem -File -Recurse -Include "*.jpg" | Update-FileNameWithNumberPadding -WhatIf` でいい感じに動いているか試す
1. `Get-ChildItem -File -Recurse -Include "*.jpg" | Update-FileNameWithNumberPadding` でリネームする
