---
title: "Azure Functions が使用するストレージアカウントの保護について"
author_name: "Shogo Ohe"
tags:
    - Function App
---
Azure Functions の利用では、ストレージアカウントの使用が必須とされており、該当のストレージアカウントの保護についてお問い合わせを頂くことがあります。この記事では、Functions が使用するストレージアカウントにおけるアクセス制限方法についてご紹介します。<br />
※この記事は、過去に公開した [Azure Functions が使用するストレージアカウントに対するアクセス制限の方法](https://azure.github.io/jpazpaas/2020/07/29/how-to-restrict-access-to-storage-used-by-azure-functions.html) のアップデートになります。

# はじめに
Azure Functions では [最大 200 インスタンス](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-scale#scale) をサポートするため、アプリケーションのコードやログは各インスタンス (仮想マシン内部) に格納せず、外部のストレージアカウントを通じてデータを共有する仕組みが採用されています。
こうしたアーキテクチャの都合上、現時点におきましては Functions をご利用頂く際はストレージアカウントの使用は避けることができません。一定の制約の中で取り得る選択肢をご案内いたします。


# 前提知識
Azure Functions の実行には、実行基盤となる App Service Plan が必要となります。App Service Plan は一種の仮想マシンであり、Azure Functions はその環境で動作するアプリケーションとして定義されます。

Azure Functions では、[Functions Host](https://github.com/Azure/azure-functions-host) と呼ばれるアプリケーション (フレームワーク) が動作しております。Functions Host が指定されたストレージからお客様の開発されたコードの読み込みと実行、Timer Trigger や Blob Trigger といったイベントの検知、Azure ポータルとの通信を担っています。

これらの通信はすべて Azure 基盤が内部的に使用するバックボーン (コントロールプレーン等) を経由するわけではありません。Functions Host はお客様のアプリケーションの一部として動作しており、お客様のアプリケーションと同じ通信経路を使用します。そのため、VNet 統合や [ネットワークセキュリティグループ](https://docs.microsoft.com/ja-jp/azure/virtual-network/network-security-groups-overview) などの影響を受けます。


Functions では、大きく分けて 3 種類のストレージが使用されます。
 1. アプリケーションコードを格納するストレージ (アプリケーション設定の [WEBSITE_CONTENTAZUREFILECONNECTIONSTRING](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#website_contentazurefileconnectionstring) で指定する Azure Files)
 2. HTTP Trigger の実行キー、Event Hub のチェックポイントを記録するストレージ (アプリケーション設定の [AzureWebJobsStorage](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#azurewebjobsstorage) で指定する Azure Files)
 3. Blob Trigger や Queue Trigger により参照されるストレージ

Azure Functions とストレージの接続、特に 1 と 2 に関して、接続元の制限を行いたいというお問い合わせをいただくことが多く、ここではその参考となる情報をご案内いたします。

前提として Azure Functions や ストレージアカウントはパブリッククラウドで提供されるサービスであり、将来的に検知される全ての脆弱性を回避できるものではありません。また、どのようなセキュリティ対策を取ったとしても新しい攻撃手法やお客様ネットワーク内からの攻撃、アプリケーションに起因する脆弱性や総当たり攻撃などに対しては無力です。そのためストレージアカウントの保護だけではなく、運用要件に合わせた統合的なセキュリティのリスク評価と受容、回避策を考慮頂く必要がございます。


## 案1. App Service Plan を使用する
[App Servie Plan を使用](https://docs.microsoft.com/ja-jp/azure/azure-functions/dedicated-plan)した場合、アプリケーションコードなどは App Service Plan が提供するファイルシステムに格納されます。具体的には、先のアプリケーション設定 "[WEBSITE_CONTENTAZUREFILECONNECTIONSTRING](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#website_contentazurefileconnectionstring)" で指定したストレージアカウントは使用しなくなりますため、アプリケーションコードのためのストレージアカウントが不要となります。

一方で、HTTP Trigger のキーや Event Hub のチェックポイントを格納するストレージ (アプリケーション設定の "[AzureWebJobsStorage](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#azurewebjobsstorage)" で指定したストレージ) は引き続き必要となります。


## 案2. ストレージアカウントからの接続で IP 制限を適用する
参考記事: [Azure Functions が使用するストレージアカウントに対するアクセス制限の方法](https://azure.github.io/jpazpaas/2020/07/29/how-to-restrict-access-to-storage-used-by-azure-functions.html) <br />
こちらにも記載がございますが、同一リージョン内の App Service, Functions から ストレージアカウントに対する接続では、データセンター内で使用されるプライベート IP アドレスが使用されます。
ストレージアカウントの [ネットワーク] > [許可するアクセス元: 選択されたネットワーク] を指定した場合、ファイアウォール にて接続を許可する IP アドレスを指定頂けますが、2021年5月現在では [プライベート IP アドレスはご指定頂くことは叶いません](https://docs.microsoft.com/ja-jp/azure/storage/common/storage-network-security?tabs=azure-portal#grant-access-from-an-internet-ip-range)。
また、App Service, Functions がストレージアカウントに接続する際に使用するプライベート IP アドレスは不定であり、特定することは困難であるため、同一リージョンに配置されたストレージアカウントではご利用頂くことは叶いません。

参考記事にも記載はございますが、Functions リソース作成時に同時に作成されるストレージを破棄し、別リージョンにストレージアカウントをアプリケーション設定にてしていただくことで接続元 IP アドレスの指定は可能になります。しかしながら、Functions として別リージョンに配置されたストレージアカウントのご利用ご利用は [パフォーマンスの観点から推奨していない](https://docs.microsoft.com/ja-jp/azure/azure-functions/storage-considerations#storage-account-location) 状況となります。


[ストレージアカウントの 診断設定](https://docs.microsoft.com/ja-jp/azure/storage/files/storage-files-monitoring?tabs=azure-portal) を使用すると、Azure Files でアクセスした日時、ファイルと接続元 IP などを調べることが出来ます。こちらを使用して Azure Functions からのアクセス状況を確認することができますが、検証を行った環境では 10.116.0.0/16 のプライベート IP からアクセスがあることが分かります。
![image-6ba438e0-df32-496c-9e3d-68bca9b03bfa.png]({{site.baseurl}}/media/2022/05/image-6ba438e0-df32-496c-9e3d-68bca9b03bfa.png)

## 案3. VNet 統合とプライベートエンドポイントを使用する
2021年2月ごろより、Functions が使用するストレージアカウントについて、[サービスエンドポイントやプライベートエンドポイントを使用した接続のサポート](https://docs.microsoft.com/ja-jp/azure/azure-functions/configure-networking-how-to#restrict-your-storage-account-to-a-virtual-network) が追加されております。
特にプライベートエンドポイントを構成し、パブリックエンドポイントからのアクセスを制限することで、ストレージアカウントに対するアクセス経路を制御することが可能となります。
- [Azure Storage のプライベート エンドポイントを使用する](https://docs.microsoft.com/ja-jp/azure/storage/common/storage-private-endpoints)

こちらの方法を使用する場合、Functions における VNet 統合は Elastic Premium プラン または Basic 以上の App Service Plan の場合ご利用いただけます。

構成手順は [ドキュメント](https://docs.microsoft.com/ja-jp/azure/azure-functions/configure-networking-how-to#restrict-your-storage-account-to-a-virtual-network) にも記載がございますが、以下のように実施します。

1. Functions を作成します。ホスティング プランは VNet 統合を使用するため、Functions Premium プラン、または Basic 以上の App Service Plan が必要となります。
2. 仮想ネットワークを作成します。サブネットは Functions 統合用とそれ以外用 (ここではストレージアカウントのプライベートエンドポイント用) の 2 つのサブネットが必要です。Functions を接続するサブネットでは最低限 /28 (11 アドレス) のレンジが必要となりますが、推奨は /24 (251 アドレス) です。必要なアドレス数などの詳細に関しては以下のドキュメントを参照してください。<br/>
  参考: [リージョンでの仮想ネットワーク統合](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-networking-options#regional-virtual-network-integration)
3. ストレージアカウントを作成します。ストレージアカウントで blob と file に対するプライベートエンドポイントを有効にします。ストレージアカウントでのプライベートエンドポイントの作成は以下のドキュメントを参照して下さい。<br />
  参考: [Azure Files ネットワーク エンドポイントの構成](https://docs.microsoft.com/ja-jp/azure/storage/files/storage-files-networking-endpoints?tabs=azure-portal)
4. 既存のストレージアカウントから、新しいストレージに対してファイルをコピーします。コピー対象は blob コンテナの azure-webjobs-hosts, azure-webjobs-secrets、[アプリケーション設定] の WEBSITE_CONTENTSHARE で指定されているファイル共有コンテナの 3 点です。
5. [Azure Functions を VNet に統合します](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-networking-options#enable-virtual-network-integration) <br />
6. Functions のストレージアカウントに対する接続文字列を書き換えます。<br/>
 [構成] > [アプリケーション設定] より AzureWebJobsStorage, WEBSITE_CONTENTAZUREFILECONNECTIONSTRING の接続文字列を 3 で作成したストレージアカウントの接続文字列に変更します。<br />
 また、関数アプリが新しいストレージ アカウントと、仮想ネットワークで経由するために、Vnet 統合設定の「コンテナストレージ」に対してチェックを入れ適用ボタンを押下します。

[3. アプリケーションの構成のルーティングを有効にする](https://learn.microsoft.com/ja-jp/azure/azure-functions/configure-networking-how-to?tabs=portal#3-enable-application-and-configuration-routing)
> コンテンツ共有ルーティングを有効にして、関数アプリが新しいストレージ アカウントと、その仮想ネットワーク経由で通信できるようにします。 前の手順と同じページで、[構成ルーティング] の下にある [コンテンツ ストレージ] を選択します。

![image-2e52809c-2333-4e57-b39d-dd6a9cb462da.png]({{site.baseurl}}/media/2022/05/image-2e52809c-2333-4e57-b39d-dd6a9cb462da.png)

[アプリケーション設定] を変更した際、自動的に Functions が再起動し、それ以降のストレージアカウントに対する通信では、Functions が使用するプライベート IP からの通信となります。
以下の図でも 100.77.12.27 という IP から 10.0.1.254 の VNet で設定したプライベート IP からのアクセスになっていることが確認できます。<br />
![image-83d5d4f1-f9dc-4469-8034-36c426fb48d5.png]({{site.baseurl}}/media/2022/05/image-83d5d4f1-f9dc-4469-8034-36c426fb48d5.png)

## 案4. VNet 統合とサービスエンドポイントを使用する

`案3. VNet 統合とプライベートエンドポイントを使用する` と同様に VNet 統合を利用して、Functions Premium プラン、または Basic 以上の App Service Plan から VNet/サブネット への通信を設定します。
ストレージアカウント側のファイアウォール設定にて、VNet 統合で使用しているサブネットを指定することでサービスエンドポイントを作成することができ、特定サブネットへのトラフィックを制限することができます。

[2. セキュリティで保護されたストレージ アカウントを作成する](https://learn.microsoft.com/ja-jp/azure/azure-functions/configure-networking-how-to?tabs=portal#2-create-a-secured-storage-account)
>特定のサブネットへのトラフィックを制限します。 関数アプリが、許可されたサブネットとネットワーク統合されていること、そのサブネットに Microsoft.Storage へのサービス エンドポイントが用意されていることを確認します。

# まとめ
Azure Functions における、ストレージアカウントの保護に関してご紹介しました。
VNet 統合が行えない都合上、Premium プラン (または、VNet 統合に対応した App Service Plan) が必要となることはネックとなりますが、ストレージアカウントをプライベートエンドポイントで保護することが可能になります。
ストレージ保護の重要性など、情報の機密性や使用したい機能などにおうじて、従量課金プラン、Premium プランの使い分けをご検討いただけますと幸いです。

<br>
<br>

2024 年 10 月 30 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>