﻿---
lab:
    title: '9: Azure Site Recovery を使用して Hyper-V VM を保護する'
    module: 'Module 9: Azure でのワークロードの管理'
---

# ラボ: Azure Site Recovery  を使用して Hyper-V VM を保護する
# 受講生用ラボ マニュアル

## ラボ シナリオ

Adatum Corporation は、長年にわたってオンプレミスのワークロードに多数の高可用性対策を実装してきましたが、その災害復旧機能は、ビジネスで要求される目標復旧時点 (RPO) および目標復旧時間 (RTO) に対処するにはまだ不十分です。既存のセカンダリ オンプレミス サイトを維持するには、多大な労力とコストが必要とされます。フェールオーバーとフェールバックの手順は、大部分が手動で行われ、十分に文書化されていません。 

これらの欠点に対処するため、Adatum エンタープライズ アーキテクチャ チームは、Azure がセカンダリ サイトのホスティングの役割を引き受けて、Azure Site Recovery の機能を探索することを決定しました。Azure Site Recovery は、物理マシンと仮想マシンで実行されているワークロードをプライマリ サイトからセカンダリ サイトに自動的かつ継続的にレプリケートします。Site Recovery は、アプリケーション データを傍受することなく、ストレージ ベースのレプリケーション メカニズムを使用します。Azure をセカンダリ サイトとして使用すると、データは Azure Storage に格納され、回復力が組み込まれ、低コストになります。ターゲット Azure VM は、複製されたデータを使用して、フェールオーバー後にハイドレートされます。Site Recovery は VMware VM に対して継続的なレプリケーションを提供し、Hyper-V VM に対して 30 秒という低いレプリケーション頻度を提供するため、目標復旧時間 (RTO) と目標復旧ポイントが最小化されます。さらに、Azure Site Recovery は、フェールオーバー プロセスとフェールバック プロセスのオーケストレーションも処理します。これらのプロセスは、大部分は自動化できます。Azure への移行に Azure Site Recovery を使用することも可能ですが、推奨されるアプローチは Azure Migrate に依存しています。

Adatum エンタープライス アーキテクチャ チームは、オンプレミスの Hyper-V 仮想マシンを Azure VM から保護するための Azure Site Recovery の使用を評価したいと考えています。

## 目的
  
このラボを終了すると、下記ができるようになります。

-  Azure Site Recover を構成する

-  テスト フェールオーバーを実行する

-  計画されているフェールオーバーを実行する

-  計画されていないフェールオーバーを実行する


## ラボ環境

Windows Server 管理者の認証資格情報

-  ユーザー名: **Student**

-  パスワード: **Pa55w.rd1234**

所要時間: 120 分


## ラボ ファイル

-  \\\\AZ303\\AllFiles\\Labs\\07\\azuredeploy30307suba.json


### 演習 0: ラボ環境の準備

この演習の主なタスクは次のとおりです。

1. Azure Resource Manager クイックスタート テンプレートを使用して Azure VM をデプロイする

1. Azure VM でネストされた仮想化を構成する


#### タスク 1: Azure Resource Manager クイックスタート テンプレートを使用して Azure VM をデプロイする

1. ラボのコンピューターから Web ブラウザーを起動し、 [Azure portal](https://portal.azure.com) に移動し、このラボで使用するサブスクリプションの所有者の役割を持つユーザーアカウントの認証情報を提供してサインインします。

1. Azure で、検索テキストボックスのすぐ右にあるツールバーアイコンを選択して、**Cloud Shell** ペインを表示します。

1. **Bash** や **PowerShell** のどちらかを選択するためのプロンプトが表示されたら、**PowerShell** を選択します。 

    >**注**: **Cloud Shell** を初めて起動し、「**ストレージがマウントされていません**」というメッセージが表示された場合は、このラボで使用しているサブスクリプションを選択し、「**ストレージの作成**」を選択します。 

1. Cloud Shell ペインのツールバーで、 **ファイルのアップロード/ダウンロード** アイコンを選択し、ドロップダウン メニューで **アップロード**を選択して、ファイル  **\\\\\AZ303\\AllFiles\Labs\\07\\azuredeploy30307suba.json** を Cloud Shell ホーム ディレクトリにアップロードします。

1. Cloud Shell ペインから次のコマンドを実行してリソースグループを作成します（ `<Azure region>`プレースホルダーを、サブスクリプションでの Azure VM のデプロイに使用可能で、ラボのコンピューターの場所に最も近い Azure リージョンの名前に置き換えます ）:

   ```powershell
   $location = '<Azure region>'
   New-AzSubscriptionDeployment `
     -Location $location `
     -Name az30307subaDeployment `
     -TemplateFile $HOME/azuredeploy30307suba.json `
     -rgLocation $location `
     -rgName 'az30307a-labRG'
   ```

      >**注意**: Azure VM をプロビジョニングできる Azure リージョンを特定するには、次を参照してください。 [**https://azure.microsoft.com/ja-jp/regions/offers/**](https://azure.microsoft.com/ja-jp/regions/offers/)

1. Azure portal で、**Cloud Shell** ウィンドウを閉じます。

1. ラボ コンピューターから別のブラウザー タブを開き、[301-nested-vms-in-virtual-network Azure クイックスタート テンプレート](https://github.com/Azure/azure-quickstart-templates/tree/master/301-nested-vms-in-virtual-network) に移動して、「**Azure にデプロイする**」 を選択します。これにより、ブラウザーは自動的に Azure portal の**入れ子になった VM を持つ Hyper-V ホスト仮想マシン** ブレードにリダイレクトされます。

1. Azure portal の**入れ子になった VM を持つ Hyper-V ホスト仮想マシン** ブレードで、次の設定を指定します (他の設定は既定値のままにします)。

    | 設定 | 値 | 
    | --- | --- |
    | サブスクリプション | このラボのために使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az30307a-labRG** |
    | ホストのパブリック IP アドレス名 | **az30307a-hv-vm-pip** |
    | 仮想ネットワーク名 | **az30307a-hv-vnet** |
    | ホスト ネットワークのインターフェイス 1 の名前 | **az30307a-hv-vm-nic1** |
    | ホスト ネットワークのインターフェイス 2 の名前 | **az30307a-hv-vm-nic2** |
    | ホスト仮想マシン名 | **az30307a-hv-vm** |
    | ホスト管理者のユーザー名 | **Student** |
    | ホスト管理者のパスワード | **Pa55w.rd1234** |

1. **入れ子になった VM を備えた Hyper-V ホスト仮想マシン** ブレードで、チェックボックス 「**上記の利用規約に同意します**」 を選択し、「**購入**」 を選択します。

    > **注**: デプロイメントが完了するのを待ちます。デプロイには約 10 分間かかります。

#### タスク 2: Azure VM でネストされた仮想化を構成する

1. Azure portal で、**仮想マシン**を検索して選択し、**仮想マシン** ブレードで **az30307a-hv-vm** を選択します。

1. **az30307a-hv-vm** ブレードで、「**ネットワーク**」 を選択します。 

1. **az30307a-hv-vm** で **| ネットワーク** ブレードで、**az30307a-hv-vm-nic1** を選択し、次に 「**受信ポート規則を追加する**」 を選択します。

    >**注**: 必ず、パブリック IP アドレスが割り当てられている **az30307a-hv-vm-nic1** の設定を変更してください。

1. 「**受信セキュリティ規則を追加**」 ブレードで、次の設定を指定し (他の設定は既定値のままにする)、「**追加**」 を選択します。

    | 設定 | 値 | 
    | --- | --- |
    | 宛先ポート範囲 | **3389** |
    | プロトコル | **任意** |
    | 名前 | **AllowRDPInBound** |

1. 「**az30307a-hv-vm**」 ブレードで、「**概要**」 を選択します。 

1. 「**az30307a-hv-vm**」 ブレードで、ドロップダウン メニューから 「**接続**」、「**RDP**」 を選択し、「**RDP ファイルのダウンロード**」 をクリックします。

1. プロンプトが表示されたら、次の認証情報を入力します。

-  ユーザー名: **Student**

-  パスワード: **Pa55w.rd1234**

1. **az30307a-hv-vm** へのリモート デスクトップ セッション内で、「サーバーマネージャー」 ウィンドウの 「**ローカル サーバー**」 をクリックし、**IE セキュリティ強化の構成**ラベルの横にある 「**オン**」 リンクをクリックし、**IE セキュリティ強化の構成**ダイアログ ボックスで両方の 「**オフ**」 オプションを選択します 。

1. **az30307a-hv-vm** へのリモート デスクトップ セッション内で、Internet Explorer を起動し、[Windows Server の評価](https://www.microsoft.com/ja-jp/evalcenter/evaluate-windows-server-2019) を参照して、Windows Server 2019 **VHD** ファイルを **F:\VHDs** フォルダーにダウンロードします (最初にこのフォルダーを作成する必要があります)。 

1. **az30307a-hv-vm** へのリモート デスクトップ セッション内で、**Hyper-V マネージャー**を起動します。 

1. **Hyper-V マネージャー** コンソールで、**az30307a-hv-vm** ノードを選択し、「**新規**」 を選択し、カスケード メニューで 「**仮想マシン**」 を選択します。これにより、**新しい仮想マシン ウィザード**が起動します。 

1. 「**新しい仮想マシン ウィザード**」 の 「**開始する前に**」 ページで、「**次へ >**」 を選択します。

1. 「**新しい仮想マシン ウィザード**」 の 「**名前と場所を指定する**」 ページで、次の設定を指定し、「**次へ >**」 を選択します。

    | 設定 | 値 | 
    | --- | --- |
    | 名前 | **az30307a-vm1** | 
    | 仮想マシンを別の場所に保存する | 選択済み | 
    | 場所 | **F:\VMs** |

    >**注**: 必ず **F:\VMs** フォルダーを作成してください。

1. 「**新しい仮想マシン ウィザード**」 の 「**世代を指定**」 ページで、「**世代 1**」 オプションが選択されていることを確認し、「**次へ >**」 を選択します:

1. 「**新しい仮想マシン ウィザード**」 の 「**メモリの割り当て**」 ページで、「**起動メモリ**」 を 「**2048**」 に設定し、「**次へ >**」 を選択します。

1. 「**新しい仮想マシン ウィザード**」 の 「**ネットワークを構成する**」 ページの 「**接続**」 ドロップダウン リストで、**NestedSwitch** を選択し、 「**次へ >**」 を選択します。

1. 「**新しい仮想マシン ウィザード**」 の 「**仮想ハード ディスクを接続する**」 ページで、オプション 「**既存の仮想ハード ディスクを使用する**」 を選択し、場所を **F:\VHDs** フォルダーにダウンロードした VHD ファイルに設定し、「**次へ >**」 を選択します。

1. 「**新しい仮想マシンウィザード**」 の 「**概要**」 ページで、「**仕上げ**」 を選択します。

1. **Hyper-V マネージャー** コンソールで、新しく作成した仮想マシンを選択し、「**開始**」 を選択します。 

1. **Hyper-V マネージャー** コンソールで、仮想マシンが実行されていることを確認し、「**接続**」 を選択します。 

1. 「**az30307a-vm1** への仮想マシン接続」 ウィンドウの 「**こんにちは**」 ページで、「**次へ**」 を選択します。 

1. 「**az30307a-vm1** への仮想マシン接続」 ウィンドウの 「**ライセンス条項**」 ページで、「**承諾**」 を選択します。 

1. 「**az30307a-vm1** への仮想マシン接続」 ウィンドウの 「**設定をカスタマイズする**」 ページで、組み込みの管理者アカウントのパスワードを **Pa55w.rd1234** に設定し、「**終了**」 を選択します。 

1. 「**az30307a-vm1** への仮想マシン接続」 ウィンドウで、新しく設定したパスワードを使用してサインインします。

1. 「**az30307a-vm1** への仮想マシン接続」 ウィンドウで、Windows PowerShell を起動し、**管理者:** で**Windows PowerShell** ウィンドウで次を実行して、コンピューター名を設定します。 

   ```powershell
   Rename-Computer -NewName 'az30307a-vm1' -Restart
   ```

### 演習 1: Azure Site Recovery Vault を作成して構成する
  
この演習の主なタスクは次のとおりです。

1. Azure Site Recovery Vault を作成する

1. Azure Site Recovery Vault を構成する


#### タスク 1: Azure Site Recovery Vault を作成する

1. **az30307a-hv-vm** へのリモートデスクトップセッション内で、Internet Explorer を起動し、 [Azure portal](https://portal.azure.com) に移動し、このラボで使用するサブスクリプションの所有者の役割を持つユーザー アカウントの認証情報を提供してサインインします。

1. Azure portal で、「**Recovery Servicesコンテナー**」を検索して選択し、「**Recovery Services コンテナー**」 ブレードで 「**+ 追加**] を選択します。

1. 「**Recovery Services コンテナーの作成**」 ブレードの 「**基本**」 タブで、次の設定を指定し (他は既定値のままにします)、「**Review + create**」 を選択します。

    | 設定 | 値 | 
    | --- | --- |
    | サブスクリプション | このラボのために使用する Azure サブスクリプションの名前 |
    | リソース グループ | 新しいリソース グループ **az30307b-LabRG** の名前 |
    | コンテナー名 | **az30307b-rsvault** |
    | 場所 | このラボの前半で仮想マシンをデプロイした Azure リージョンの名前 |

1. 「**Recovery Services コンテナーの作成**」 ブレードの 「**Review + create**」 タブで、「**作成**」 を選択します。

    >**注**: 既定では、ストレージ レプリケーション タイプの既定の構成は Geo 冗長 (GRS) に設定され、ソフト削除が有効になっています。ラボでこれらの設定を変更してプロビジョニング解除を簡素化しますが、運用環境で使用する必要があります。


#### タスク 2: Azure Site Recovery Vault を構成する

1. Azure portal で、「**Recovery Services コンテナー**」 を検索して選択し、「**Recovery Services コンテナー**」 ブレードで、「**az30307b-rsvault**」 を選択します。

1. 「**az30307b-rsvault**」 ブレードで 「**プロパティ**」 を選択します。 

1. **az30307b-rsvault** で **| プロパティ** ブレードで、「**バックアップ構成**」 ラベルの下の 「**更新**」 リンクを選択します。

1. 「**バックアップ構成**」 ブレードで、「**ストレージ レプリケーションの種類**」 を 「**ローカル冗長**」 に設定し、「**保存**」 を選択して 「**バックアップ構成**」 ブレードを閉じます。

    >**注**: アイテムの保護を開始すると、ストレージ レプリケーションの種類は変更できません。

1. **az30307b-rsvault** で **| プロパティ** ブレードで、「**セキュリティ設定**」 ラベルの下の 「**更新**」 リンクを選択します。

1. 「**セキュリティ設定**」 ブレードで、「**ソフト削除**」 を 「**無効**」 に設定し、「**保存**」 を選択して 「**セキュリティ設定**」 ブレードを閉じます。


### 演習 2: Azure Site Recovery Vault を使用して Hyper-V 保護を実装する
  
この演習の主なタスクは次のとおりです。

1. ターゲットの Azure 環境を実装する

1. Hyper-V 仮想マシンの保護を実装する

1. Hyper-V 仮想マシンのフェールオーバーを実行する

1. ラボにデプロイした Azure リソースを削除する


#### タスク 1: ターゲットの Azure 環境を実装する

1. Azure portal で、**仮想ネットワーク**を検索して選択し、「**仮想ネットワーク**」 ブレードで、「**+ 追加**」 を選択します。

1. 「**仮想ネットワークを作成する**」 ブレードの 「**基本**」 タブで、次の設定を指定し (他の設定は既定値のままにします)、「**次へ: IP アドレス**」:

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボのために使用する Azure サブスクリプションの名前 |
    | リソース グループ | 新しいリソース グループ **az30307c-LabRG** の名前 |
    | 名前 | **az30307c-dr-vnet** |
    | リージョン | このラボの前半で仮想マシンをデプロイした Azure リージョンの名前 |

1. 「**仮想ネットワークを作成する**」 ブレードの 「**IP アドレス**」 タブで、「**IPv4 アドレス空間**」 テキスト ボックスに「**10.7.0.0/16**」と入力し、「**+ サブネットを追加**」 を選択します。

1. 「**サブネットを追加**」 ブレードで、次の設定を指定し (他の設定は既定値のままにします)、「**追加**」 を選択します。

    | 設定 | 値 |
    | --- | --- |
    | サブネット名 | **subnet0** |
    | サブネット アドレス範囲 | **10.7.0.0/24** |

1. 「**仮想ネットワークを作成する**」 ブレードの 「**IP アドレス**」 タブに戻り、「**Review + create**」 を選択します。

1. 「**仮想ネットワークを作成する**」 ブレードの 「**Review + create**」 タブで、「**作成する**」 を選択します。

1. Azure portal で、**仮想ネットワーク**を検索して選択し、「**仮想ネットワーク**」 ブレードで、「**+ 追加**」 を選択します。

1. 「**仮想ネットワークを作成する**」 ブレードの 「**基本**」 タブで、次の設定を指定し (他の設定は既定値のままにします)、「**次へ: IP アドレス**」:

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボのために使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az30307c-labRG** |
    | 名前 | **az30307c-test-vnet** |
    | リージョン | このラボの前半で仮想マシンをデプロイした Azure リージョンの名前 |

1. 「**仮想ネットワークを作成する**」 ブレードの 「**IP アドレス**」 タブで、「**IPv4 アドレス空間**」 テキスト ボックスに「**10.7.0.0/16**」と入力し、「**+ サブネットを追加**」 を選択します。

1. 「**サブネットを追加**」 ブレードで、次の設定を指定し (他の設定は既定値のままにします)、「**追加**」 を選択します。

    | 設定 | 値 |
    | --- | --- |
    | サブネット名 | **subnet0** |
    | サブネット アドレス範囲 | **10.7.0.0/24** |

1. 「**仮想ネットワークを作成する**」 ブレードの 「**IP アドレス**」 タブに戻り、「**Review + create**」 を選択します。

1. 「**仮想ネットワークを作成する**」 ブレードの 「**Review + create**」 タブで、「**作成する**」 を選択します。

1. Azure portalで、「**ストレージ アカウント**」 を検索して選択し、「**ストレージ アカウント**」 ブレードで、「**+ 追加**」 を選択します。

1. 「**ストレージ アカウントを作成する**」 ブレードの 「**基本**」 タブで、次の設定を指定します (他の設定は既定値のままにします)。

    | 設定 | 値 | 
    | --- | --- |
    | サブスクリプション | このラボのために使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az30307c-labRG** |
    | ストレージ アカウント名 | 文字と数字で構成される、長さが 3 ? 24 のグローバルに一意の名前 |
    | 場所 | このタスクの前半で仮想ネットワークを作成した Azure リージョンの名前 |
    | パフォーマンス | **Standard** |
    | アカウントの種類 | **StorageV2 (汎用 v2)** |
    | レプリケーション | **ローカル冗長ストレージ (LRS)** |

1. 「**ストレージ アカウントを作成する**」 ブレードの 「**基本**」 タブで、「**Review + create**」 を選択します。

1. 「**ストレージ アカウントを作成する**」 ブレードの 「**Review + create**」 タブで、「**作成**」 を選択します。


#### タスク 2: Hyper-V 仮想マシンの保護を実装する

1. 「**az30307a-hv-vm**」 へのリモート デスクトップ セッション内の、Azure portal の 「**az30307b-rsvault**」 ブレードの 「**はじめに**」 セクションで、「**Site Recovery**」 を選択します

1. **az30307b-rsvault** で **| Site Recovery** ブレードの 「**オンプレミスのマシンの場合**」 セクションで、「**インフラストラクチャの準備**」 を選択します。 

1. 「**保護の目標**」 ブレードで、次の設定を選択し (他は既定値のままにします)、「**OK**」 を選択します。

    | 設定 | 値 | 
    | --- | --- |
    | ご使用のマシンがある場所 | **オンプレミス** |
    | マシンをどこに複製しますか | **Azure** |
    | 移行を実行していますか？ | **いいえ** |
    | Virtual Machines は仮想化されていますか | **はい、Hyper-V を使用します** |
    | System Center VMM を使用して Hyper-V ホストを管理していますか?  | **いいえ** |

1. **導入計画**ブレードの **導入計画を完了しましたか?** というラベルの付いたドロップダウン リストで、「**はい、完了しました**」 を選択し、「**OK**」 を選択します

1. 「**ソースの準備**」 ブレードで、「**+ Hyper-V サイト**」 を選択します。 

1. 「**Hyper-V サイトを作成する**」 ブレードの 「**名前**」 テキスト ボックスに「**az30307b Hyper-V site**」と入力し、「**OK**」 を選択します:

1. 「**ソースを準備する**」 ブレードに戻り、「**+ Hyper-V サーバー**」 を選択します。 

    >**注**: ブラウザー ページを更新する必要がある場合があります。 

1. 「**サーバーの追加**」 ブレードで、オンプレミスの Hyper-V ホストを登録する手順のステップ 3 の 「**ダウンロード**」 リンクを選択して、Microsoft Azure Site Recovery プロバイダーをダウンロードします。

1. プロンプトが表示されたら、**AzureSiteRecoveryProvider.exe** を起動します。**Azure Site Recovery Provider のセットアップ (Hyper-V サーバー)** ウィザードが開始されます。

1. **Microsoft Update** ページで、「**オフ**」、「**次へ**」 を選択します。

1. **プロバイダーのインストール** ページで、「**インストール**」 を選択します。

1. Azure に切り替え、「**サーバーを追加**」 ブレードで、オンプレミスの Hyper-V ホストを登録する手順のステップ 4 の 「**ダウンロード**」 ボタンを選択して、Vault 登録キーをダウンロードします。プロンプトが表示されたら、登録キーを**ダウンロード**フォルダーに保存します。

1. **プロバイダーのインストール** ページに切り替え、「**登録**」 を選択します。これにより、**Microsoft Azure Site Recovery 登録ウィザード**が起動します。

1. **Microsoft Azure Site Recovery 登録ウィザード**の**コンテナーの設定**ページで、「**参照**」 を選択し、**ダウンロード** フォルダーに移動し 、コンテナー認証情報ファイルを選択し、「**開く**」 を選択します。

1. **Microsoft Azure Site Recovery 登録ウィザード**の**コンテナ―の設定**ページに戻り、「**次へ**」 を選択します。

1. **Microsoft Azure Site Recovery 登録ウィザード**の**プロキシ設定**ページで、既定の設定を受け入れ、「**次へ**」 を選択します。

1. **Microsoft Azure Site Recovery 登録ウィザード**の**登録**ページで、**仕上げ**を選択します。

1. Azure portal を表示しているページを更新し、「**ソースの準備**」 ブレードに到達するまで一連の手順を繰り返します。Hyper-V サイトと Hyper-V サーバーの両方が追加されていることを確認し、「**OK**」 を選択します。

1. 「**目標**」 ブレードで 「**+ ストレージアカウント**」 を選択し、この演習の最初のタスクで作成したストレージ アカウントを選択します。

1. 「**目標**」 ブレードで 「**+ ネットワーク**」 を選択し、この演習の最初のタスクで作成した仮想ネットワークを選択します。

1. 「**複製ポリシー**」 ブレードで、「**+ 作成と関連付**」 を選択します。 

1. 「**作成と関連付け**」 ブレードで次の設定を指定し (他の設定は既定値のままにします)、「**OK**」 を選択します。

    | 設定 | 値 |
    | --- | --- |
    | 名前 | **az30307c レプリケーション ポリシー** |
    | コピー頻度 | **30 秒** |

1. **インフラストラクチャの準備** ブレードに戻る場合は、**「OK」** を選択します。

1. **az30307b-rsvault** に戻る **| オンプレミス マシンと Azure VM の場合** セクションの **Site Recovery** ブレード、 **ステップ 1:**を選択します**アプリケーションをレプリケートする**。 

1. **ソース** ブレードで、既定の設定値を受け入れてから、**OK** をクリックします。

    >**注**: **OK** の場合、ドロップダウンリストの項目を選択解除して再選択します。

1. **目標** ブレードで次の設定を指定し（他の設定は既定値のままにします）、**OK**を選択します

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボのために使用する Azure サブスクリプションの名前 |
    | フェールオーバー後のリソース グループ | **az30307c-labRG** |
    | フェールオーバー後のデプロイ モデル | **Resource Manager** |
    | ストレージ アカウント | この演習の最初のタスクで作成したストレージ アカウントの名前 |
    | Azure ネットワーク | 選択したマシンに今すぐ校正 |
    | フェールオーバー後の Azure ネットワーク | **az30307c-dr-vnet** |
    | サブネット | **subnet0 (10.7.0.0/24)** |

1. **仮想マシンの選択** ブレードで、 **az30307a-vm1** を選択し、 **OK**を選択します。

1. 「**プロパティの構成**」 ブレードの 「**既定**」 行と 「**OS タイプ**」 列で、ドロップダウンリストから「**Windows**」 を選択し、「**OK**」 を選択します。

1. **レプリケーション設定の構成** ブレードで、デフォルト設定を承諾し、**OK**を選択します。

1. **レプリケーションの有効** ブレードに戻り、**レプリケーションの有効**を選択します。


#### タスク 3: Azure VM レプリケーション設定の確認

1. Azure portal で、**az30307b-rsvault** ブレードに戻り、**レプリケートされたアイテム**を選択します。 

1. **az30307b-rsvault** で **| レプリケートされたアイテム** ブレードで、「**az30307a-vm1**」 仮想マシンを表すエントリがあることを確認し、その 「**レプリケーションの正常性**」 が「**正常**」と表示されていることと、その 「**状態**」 が「**保護を有効化中**」と表示されていることを確認します。

    > **注**: 「**az30307b-rsvault - レプリケートされたアイテム**」 ブレードに 「**az30307a-vm1**」 エントリが表示されるまで数分待機する必要がある場合があります。

1. 「**az30307b-rsvault - レプリケートされたアイテム**」 ブレードで、「**az30307a-vm1**」 エントリを選択します。

1. 「**az30307a-vm1**」 のレプリケートされたアイテム ブレードで、「**正常性と状態**」、「**フェールオーバーの準備**」、「**最新の回復ポイント**」、「**インフラストラクチャ ビュー**」 セクションを確認します。「**計画されたフェールオーバー**」、「**フェールオーバー**」、および 「**テスト フェールオーバー**」 ツールバー アイコンに注意してください。

1. ステータスが 「**保護**」 に変わるまで待ちます。これはおよそ 15 分間かかるかもしれません。

1. **az30307a-vm1** のレプリケーション アイテム ブレードで、**「最新の復旧ポイント」**を選択し、**最新のクラッシュ整合性** と**最新のアプリ整合性**復旧ポイントを確認します。


#### タスク 3: Hyper-V 仮想マシンのフェールオーバーを実行する

1. **az30307a-vm0** へのリモート デスクトップ セッション内で、Azure portal を表示しているブラウザーの画面の **az30307a-vm1** レプリケート アイテム ブレードで、**「テスト フェールオーバー」** を選択します。 

1. **「フェールオーバーをテストする」** ブレードで次の設定を指定し（他の設定は既定値のままにします）、**「OK」** を選択します。

    | 設定 | 値 |
    | --- | --- |
    | 復旧ポイントを選択してください | 既定値の選択 | 
    | Azure Virtual Network | **az30307c-test-vnet** | 


1. Azure portal で、**「az30307b-rsvault」** ブレードに戻り、**「Site Recovery ジョブ」** を選択します。**フェールオーバーのテスト**ジョブのステータスが **「成功」** と表示されるまで待ちます。

1. Azure portal で、**仮想マシン**を検索して選択し、「**仮想マシン**」 ブレードで、新しくプロビジョニングされた仮想マシン **az30307a-vm1-test** を表すエントリを確認します。

1. Azure portal で、**az30307a-vm1** のレプリケートされたアイテムのブレードに戻り、「**クリーンアップ テスト フェールオーバー**」 を選択します。

1. **「フェールオーバーのクリーンアップをテスト」** ブレードで、チェックボックスの **「テスト完了」** を選択します。**テスト フェールオーバー 仮想マシンを削除する** を選択し、**「OK」** をクリックします。

1. テスト フェールオーバー クリーンアップ ジョブが完了したら、**az30307a-vm1** の複製されたアイテムのブレードを表示するブラウザーページを更新します。計画的および計画外のフェールオーバーを実行するオプションがあることに注意してください。

1. **az30307a-vm1** のレプリケート アイテムブレードで、**「計画的なフェールオーバー」** を選択します。 

1. **「計画的なフェールオーバー」** ブレードで 、フェールオーバー方向の設定は、すでに設定されており、変更できないことに注意してください。 

1. **「計画的なフェイルオーバー」** ブレードを閉じ、**az30307a-vm1** レプリケート アイテム ブレードで **「フェールオーバー」** を選択します。 

1. **「フェールオーバー」** ブレードで、データ損失の可能性を最小限に抑えることを目的とした利用可能なオプションを確認してください。 

1. 「**フェールオーバー**」 ブレードを閉じます。


#### タスク 4: ラボにデプロイした Azure リソースを削除する

1. **az30307a-vm0** へのリモート デスクトップ セッション内の Azure portal を表示しているブラウザーの画面で、Cloud Shell ペイン内で PowerShell セッションを起動します。

1. 「Cloud Shell」 ウィンドウから次のコマンドを実行して、この演習で作成したリソース グループを一覧表示します。

   ```powershell
   Get-AzResourceGroup -Name 'az30307*'
   ```

    > **注**: このラボで作成したリソース グループのみが出力に含まれていることを確認します。このグループは、このタスクで削除されます。

1. 「Cloud Shell」 ウィンドウから次を実行して、このラボで作成したリソース グループを削除します

   ```powershell
   Get-AzResourceGroup -Name 'az30307*' | Remove-AzResourceGroup -Force -AsJob
   ```

1. 「Cloud Shell」 ウィンドウを閉じます。