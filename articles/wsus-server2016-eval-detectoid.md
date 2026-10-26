---
title: "Server 2016評価版でWSUSの更新が検出されない原因をメタデータから追った"
emoji: "🔎"
type: "tech"
topics: ["windowsserver", "wsus", "powershell", "windowsupdate"]
published: true
---

## はじめに

Windows Server 2016のドメインコントローラーに、WSUS経由でセキュリティ更新を適用するための検証をしていました。

ところが、検証用の評価版OSでは、2025年7月の累積更新プログラムは適用できるのに、8月の累積更新プログラムが検出されません。WSUSでは承認済みで、コンテンツのダウンロードも完了しています。

調査の結果、8月LCUに追加された前提条件にたどり着きました。その条件はOSの`EditionID`を判定しており、検証端末の`ServerStandardEval`が許可対象に含まれていませんでした。

さらに、当初検出できずに困っていた2026年9月LCU（KB5123099）についても、同じ判定条件を直接の前提条件として参照していることを確認しました。

この記事では、2025年7月・8月の比較で原因を追い、2026年9月LCUにも同じ阻害条件があることを確認するまでの過程をまとめます。

:::message
本記事は2026年10月26日〜27日に確認した検証記録です。対象はWindows Server 2016 Standard Evaluationと、取得時点の更新メタデータです。Microsoftが評価版を意図的に除外したのか、不具合なのかは確認できていません。将来のメタデータや、すべての更新に同じ結論が当てはまるとは限りません。
:::

## 検証環境と用語

本番ネットワークから分離したHyper-V上のAD環境で検証しました。本番DCを直接変更した検証ではありません。

| 項目 | 内容 |
|---|---|
| 更新対象 | Windows Server 2016 Standard Evaluationのメンバーサーバー |
| 当初の問題発生端末 | 同じ評価版の検証用DC（LAB-DC01） |
| WSUSサーバー | Windows Server 2016、WID使用 |
| WSUS接続 | HTTP 8530、GPOで参照先を指定 |
| 承認範囲 | 検証端末1台だけを所属させた専用グループ |
| 操作環境 | Windows PowerShell 5.1 |
| 検証開始時のビルド | 14393.693 |

本文で使う用語は次のとおりです。

- **LCU**：Latest Cumulative Update。累積更新プログラム。
- **SSU**：Servicing Stack Update。更新プログラムを適用する基盤の更新。
- **WUA**：Windows Update Agent。更新の検索や適用判定などを行うクライアント側の仕組み。
- **EditionID**：Windowsのエディションを表す内部の文字列。通常版Standardは`ServerStandard`、今回の評価版は`ServerStandardEval`でした。

## 1. まず7月LCUのWSUS配布を成功させる

当初の目的は、検証用DCに2026年9月LCU（KB5123099）を配布することでした。しかし、承認済みでも`NotApplicable`となったため、比較の基準を作る目的で過去の更新へ戻って検証しました。

最初から新しいLCUだけを追うと、WSUSの構築に問題があるのか、更新固有の問題なのかを分けられません。

そこで、古いビルドのWindows Server 2016を用意し、次の順序で適用しました。

1. SSU：KB5062799
2. 7月LCU：KB5062560
3. 再起動
4. OSビルド、CBSパッケージ、WSUSへの報告を確認

結果は次のとおりです。

```text
OS Build                  : 14393.8246
KB5062560 CBS State        : Installed
KB5062799 WSUS ClientState : Installed
KB5062560 WSUS ClientState : Installed
```

これで、少なくともこの更新の組み合わせでは、WSUSの検出・ダウンロード・インストール・完了報告まで動作することを確認できました。

なお、検証時点では7月LCUも置き換え済みでした。対象UpdateIDを指定し、`IncludePotentiallySupersededUpdates = $true`にした検索で検出を確認しています。最初の通常検索にはコンテンツ取得のタイミングも関係した可能性があるため、「通常検索で0件だった理由」をこの設定だけに断定していません。

## 2. 8月LCUは配布準備が完了しても検出されない

次に、8月LCUのKB5063871を同じ専用グループだけに承認しました。

```text
UpdateID         : f674ff1b-8872-4c09-8122-46e7b067cf88
Revision         : 200
State            : Ready
Approval         : Install
IsDeclined       : False
IsSuperseded     : True
IsLatestRevision : True
```

コンテンツ取得完了後、通常検索と、置き換え済み更新を含めた対象ID指定検索を比較しました。

| 検索方法 | ResultCode | 検出件数 |
|---|---|---|
| 通常検索 | 2（成功） | 0 |
| 対象ID指定、置き換え済み更新を含む | 2（成功） | 0 |

検索に警告はなく、WSUSの報告時刻が更新された後も、対象端末の状態は`NotApplicable`でした。

つまり、「検索がエラーになった」のではなく、「検索は成功したが、対象の更新が提示されない」状態です。

## 3. DISMでは適用可能で、実際にインストールできた

WSUSに保存されたKB5063871のSelfContained CABを検証端末へ取得し、ハッシュを照合してからDISMで情報を確認しました。

```text
Package Identity : Package_for_RollupFix~31bf3856ad364e35~amd64~~14393.8330.1.10
Applicable       : Yes
State            : Not Present
```

WUAでは対象外なのに、DISMでは適用可能です。

検証用VMでこのCABを手動適用すると、DISMは終了コード`3010`を返しました。再起動後は次の状態になりました。

```text
OS Build         : 14393.8330
CBS PackageState : Installed
CBSRebootPending : False
WURebootRequired : False
```

しかし、その後にWUAで適用済みの8月LCUを検索しても0件でした。WSUSへ新しい状態が報告された後も、表示は`NotApplicable`のままです。

ここで、**パッケージ本体を適用できることと、WUAが更新を対象として扱うことは、分けて確認する必要がある**と分かりました。

この手動適用は検証のための操作です。本番での標準運用として採用したわけではなく、手動適用によってWUAの検出が復旧したわけでもありません。

## 4. 7月と8月のメタデータを比較する

次に、更新メタデータの前提条件を比較しました。

最初に実施したWSUS APIの取得では、前提更新・子更新の件数がどちらも0件でした。しかし、その後に取得したXMLには両方の関係が存在しました。このため、最初のAPI結果だけを根拠に「前提条件がない」とは判断できませんでした。

今回の検証ではSUSDBに対してSELECTのみを実施し、`tbUpdate`、`tbRevision`、`tbXml`の関係をたどってXMLを取得しました。DBの更新や、配布メタデータの編集はしていません。

取得した圧縮データの先頭は、GZipではなくCABの署名でした。

```text
4D-53-43-46 = MSCF
```

`expand.exe`で展開したデータはUTF-16 LEで、Type1には`&lt;UpdateIdentity ...`のようにエスケープされたXML断片が入っていました。文字コードを正しく指定して読み取り、エスケープを1回解除して比較しました。

:::message
SUSDBの内部構造を前提とした調査です。一般的な運用手順としてDBを直接操作することを勧めるものではありません。また、本記事のType0・Type1という表記は、取得した`RootElementType`の値です。
:::

比較の結果、8月LCUにだけ次の前提条件が追加されていました。

```xml
<UpdateIdentity UpdateID="00729d5c-44f5-4491-9080-44cecd9e6df8" />
```

| 更新 | 今回比較したRevision | この前提条件 |
|---|---|---|
| KB5062560（7月） | 200 | なし |
| KB5063871（8月） | 200 | あり |

子更新のIDにも差分がありましたが、まず新しく追加された前提条件を追いました。

## 5. 追加された条件はEditionIDの判定だった

追加されたIDのメタデータを取得すると、Revision 202のDetectoidでした。Detectoidは、通常のインストール対象パッチとは異なり、前提条件の評価などに使われる判定用の更新です。

取得したメタデータでは`UpdateType="Detectoid"`、`ExplicitlyDeployable="false"`となっていました。

判定対象は、次のレジストリ値です。

```text
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
EditionID
```

`IsInstalled`の下に`Or`があり、`RegSz`の`Comparison="EqualTo"`で、次の6種類を比較していました。

| 許可されるEditionID |
|---|
| ServerEssentialsA |
| ServerDatacenter |
| ServerStandard |
| ServerHyper |
| ServerStorageStandard |
| ServerStorageWorkgroup |

一方、検証端末の値は次のとおりです。

```text
EditionID : ServerStandardEval
```

**どの値にも一致しません。**

ここでの`IsInstalled`は「このDetectoidというパッチを手動インストールすればよい」という意味ではありません。このDetectoidでは、指定したエディションに一致するかどうかを判定しています。

追加前提条件に評価版が含まれていないことは、今回の未検出を説明する具体的な根拠になりました。

## なぜ7月は成功し、8月は対象外になるのか

確認結果を並べると、次のようになります。

| 事象 | 確認した条件との関係 |
|---|---|
| 7月LCUはWSUS経由で適用成功 | 今回のエディション判定が前提条件にない |
| 8月LCUは承認済みでも検出されない | 追加された前提条件に`ServerStandardEval`がない |
| 8月LCUはDISMで手動適用できる | パッケージ自体の適用可否とWUAの判定が異なる |
| 手動適用後もWSUSでは対象外 | EditionIDが変わらず、追加条件を満たさない |

このため、本検証環境ではエディション条件の不一致が、8月LCUをWUAで検出するうえでの阻害要因と判断しました。

ただし、EditionIDの書き換えや通常版への変換によって検出が復旧するところまでは検証していません。「この条件を満たせば、ほかの条件も含めて必ず更新できる」とは言い切れません。

## 6. 2026年9月LCUにも同じ条件があるか確認する

2025年8月LCUだけの問題か、それとも当初の2026年9月LCU未検出にも関係するのかを確認するため、KB5123099のメタデータも取得しました。

WSUS上の対象は次の状態でした。

```text
Title            : 2026-09 x64 ベース システム用 Windows Server 2016 の累積更新プログラム (KB5123099)
UpdateID         : 8bab8bed-42e4-4562-b3c7-c2af16f0153d
Revision         : 200
State            : Ready
IsDeclined       : False
IsSuperseded     : False
IsLatestRevision : True
```

そのType1の`Relationships/Prerequisites`には、同じIDが直接記載されていました。以下は該当する参照の抜粋です。

```xml
<UpdateIdentity UpdateID="00729d5c-44f5-4491-9080-44cecd9e6df8" />
```

XMLを検査したスクリプトの結果も、直接の前提条件としての存在を示しています。

```text
KB                       : 5123099
Revision                 : 200
DetectoidID              : 00729d5c-44f5-4491-9080-44cecd9e6df8
DirectPrerequisite       : True
FoundWithinPrerequisites : True
```

同じSUSDB内で、このIDの最新リビジョンは先に確認したRevision 202です。そのエディション条件に、検証用DCの`ServerStandardEval`は一致しません。

これにより、**2026年9月LCUにも、評価版を対象から外す同じ前提条件が存在する**ことを確認できました。今回の条件は、当初のLAB-DC01での未検出を説明する阻害要因でもあります。

ただし、2026年9月LCUの確認範囲は、2025年8月LCUとは異なります。

| 更新 | 同じDetectoidの直接参照 | 検証端末と確認結果 |
|---|---|---|
| 2025年7月：KB5062560 | なし | LAB-PATCH16でWSUS適用・Installed報告まで成功 |
| 2025年8月：KB5063871 | あり | LAB-PATCH16でWSUSは対象外。DISM手動適用成功 |
| 2026年9月：KB5123099 | あり | LAB-DC01でWSUSは対象外。DISMの情報取得はApplicable＝Yes。手動適用成功は未確認 |

:::message
2026年9月LCUは、当初配布しようとしていた更新です。本記事では2026年10月分や調査時点の最新版の確認はしていません。また、2025年8月と2026年9月の両方で同じ条件を確認したことは、その間のすべてのLCUを個別に検証したことを意味しません。
:::

## 評価版を意図的に除外したのか

取得したDetectoidのタイトルには`EOL Detectoid`という文字がありました。しかし、タイトルだけでは、Microsoftが評価版の更新提供を意図的に終了したとは判断できません。

分かったのは、取得時点の条件に評価版が含まれていないという事実です。意図した対象制限なのか、条件の記載漏れなのかは、公式な説明を確認できていません。

また、今回取得したDetectoidのRevision 202には2025年10月のCreationDateが記録されていました。この記事は2026年に取得したメタデータの比較であり、**2025年8月の公開当初も完全に同じ判定条件だったと証明するものではありません。**

## EditionIDを書き換えれば解決するのか

本検証では書き換えていません。

レジストリの文字列を変更することと、評価版OSを正式に通常版へ変換することは別です。文字列だけを変更しても、OS内部の構成やライセンス状態が通常版になるわけではありません。

本番向けの解決策としては扱わず、本番と同じ通常版Windows Server 2016で検証を進める方針です。

## 同様の事象で最初に確認したいこと

エディションの確認だけなら、対象端末で次の参照専用スクリプトを実行できます。Windows PowerShell 5.1向けです。更新検索や設定変更は行いません。

```powershell
& {
    $ErrorActionPreference = 'Stop'

    $OS = Get-CimInstance -ClassName Win32_OperatingSystem
    $Version = Get-ItemProperty -LiteralPath (
        'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion'
    )

    $AllowedEditions = @(
        'ServerEssentialsA'
        'ServerDatacenter'
        'ServerStandard'
        'ServerHyper'
        'ServerStorageStandard'
        'ServerStorageWorkgroup'
    )

    [pscustomobject]@{
        ComputerName = $env:COMPUTERNAME
        OS           = $OS.Caption
        EditionID    = $Version.EditionID
        FullBuild    = '{0}.{1}' -f $Version.CurrentBuild, $Version.UBR
        MatchesObservedEditionRule = (
            $AllowedEditions -ccontains [string]$Version.EditionID
        )
    } | Format-List
}
```

`MatchesObservedEditionRule`は、この記事で確認した6種類との一致を示すだけです。Windows Update自身の適用判定結果ではなく、更新全体の適用可否やライセンス認証の状態を保証するものでもありません。

## 今回の検証で得られたこと

WSUSの検証では、承認、通信、GPO、コンテンツ取得を確認しがちです。しかし、検索が正常終了しても更新が提示されない場合は、更新メタデータ側の条件も関係します。

今回、2025年7月LCUでは配布からInstalled報告まで成功し、8月LCUでは評価版を含まない前提条件が見つかりました。さらに、当初の対象だった2026年9月LCUでも、同じDetectoidの直接参照を確認できました。これにより、WSUSの基本機能の検証と、更新メタデータの対象判定の問題を切り分けられました。

今後は、本番DCのEditionIDを確認し、通常版Windows Server 2016で検出・適用・状態報告まで検証する必要があります。通常版なら今回の条件には一致しますが、現時点で本番での成功まで確認したわけではありません。

同じ「Windows Server 2016」でも、**評価版と通常版で更新の検出条件が同じとは限らない**。今回の調査で得た、検証環境を設計するうえでの注意点です。

## 参考資料

- [Microsoft Learn：Update Applicability Rules](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/bb902473(v=vs.85))
- [Microsoft Learn：How Windows Update works](https://learn.microsoft.com/en-us/windows/deployment/update/how-windows-update-works)
- [Microsoft Learn：[MS-WUSP] Populating the Data Model](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-wusp/7b42ccc2-770b-4452-a0f8-e731474ad619)

これらは適用条件やDetectoidの仕組みに関する参考資料です。本記事の評価版除外について、Microsoftが不具合・仕様のいずれかと認定したことを示す資料ではありません。