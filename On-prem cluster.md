* AKS on Azure Stack Hub
* AKS on Azure Stack HCI
* EKS Anywhere
* EKS Outposts
* GCP Anthos
* Openshift
* SUSE Rancher RKE
* vmware tanzu
# Azure
## Azure Stack Hub
[[Azure Stack Hub Deep Dive]]
基本上就是 Azure 直接在你家幫你建一套 local 版 Azure 
* Azure Stack Hub 是賣一整套 hardware + software 的，由當地的 hardware partner ([Pricing 的 Availability 裡面有列](https://azure.microsoft.com/en-us/pricing/details/azure-stack/hub)，包括 Saudi Arabia) 直接把 4-16 servers per rack (scale unit) 的 hardware 直接送到你的 data center 跟你點交。![scale unit](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-overview/azure-stack-integrated-system.svg?view=azs-2108)
* 這些機器可以在 connected 或[完全 disconnected](https://docs.microsoft.com/en-us/azure-stack/operator/azure-stack-disconnected-deployment?view=azs-2108) 的模式下運作，基本上就是為了 target 一些特殊的 regulation 的需求。
* 會有一個跟 Azure Portal 長得八成像的 Web GUI，基本上上面就是提供 Azure 的服務讓你可以在 on-prem 環境下用。 ![](https://docs.microsoft.com/en-us/azure-stack/operator/media/tutorial-offer-services/1-create-resource-offer.png?view=azs-2108)
* [計費模式跟 Azure 一樣是 pay-as-you-go。如果要跑 disconnected 的話，可以根據 physical cores 一次 subscribe 一整年的 plan](https://azure.microsoft.com/en-us/pricing/details/azure-stack/hub)
* data 都是存在 local 的，user 可以根據需求決定要不要上傳 azure （例如傳 metric 到 azure 做 monitoring）
* 可以用 Azure Stack Development Kit ([ASDK](https://github.com/Azure-Samples/Azure-Stack-Hub-Foundation-Core/tree/master/Tools/ASDKscripts)) 在 local 跑一個試用版的 Stack Hub 以評估使用需求。
* 有兩種方式 deploy Kubernetes
	* [AKS Engine](https://docs.microsoft.com/en-us/azure-stack/user/azure-stack-kubernetes-aks-engine-overview)
		* open source https://github.com/Azure/aks-engine
		* CLI tool
		* 用 Azure Resource Manager 的 template 去 deploy 在像是 Azure VM 等等比較 low-level 的 resource 上，算是早期還沒有提供直接在 Azure Portal 上可以直接開 AKS 的產物
	* [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure-stack/user/aks-overview?view=azs-2108)
		* PREVIEW only
		* CLI/GUI tool
		* 這個才是真的跟 Azure cloud 上的 AKS 一樣的對等的東西，AKS Engine 已經對 cloud 的用戶 deprecated 了（但還會持續 support AKS Engine on Azure Stack Hub 直到這個 AKS on Azure Stack Hub 成熟）
		* [有些 Azure AKS 的 feature 因為 physical 環境原因所以不會 support](https://docs.microsoft.com/en-us/azure-stack/user/aks-overview?view=azs-2108#feature-comparison)

## Azure Stack HCI
* Azure Stack HCI 是賣 virtualization solution，也就是 Azure HCI OS。你可以 (1) 跟 hardware partner 買 [OS built-in 的機器](https://aka.ms/AzureStackHCICatalog) 或 (2) 用 validated server 然後自己裝 Azure HCI OS（會收費）
* local 的管理是用一個叫 Windows Admin Center 的 tools ![](https://docs.microsoft.com/en-us/azure-stack/hci/manage/media/manage-vm/new-vm.png#lightbox)
* [有 connectivity 的 requirement](https://docs.microsoft.com/en-us/azure-stack/hci/overview#what-you-need-for-azure-stack-hci) ，不過不是很 strict，要能夠至少每 30 天連到 Azure
* [一個 HCI cluster 最少 2 nodes 最多 16 nodes](https://docs.microsoft.com/en-us/azure-stack/hci/concepts/system-requirements#server-requirements)，不過 HCI clusters 可以用 [cluster sets](https://docs.microsoft.com/en-us/azure-stack/hci/deploy/cluster-set) 再合併起來變成 HCI platform，所以還是可以有數百個 nodes
* [on-premises Kubernetes with AKS on Azure Stack HCI](https://docs.microsoft.com/en-us/azure-stack/aks-hci/overview) ![](https://docs.microsoft.com/en-us/azure-stack/aks-hci/media/aks-hci-deployment.gif)
* _TODO: HCI cluster 都會有 max node 數量上限，那 HCI cluster 跟 AKS cluster 的 node group 之間的 mapping 是什麼？AKS cluster 會因此有 capacity 上限嗎？_

## Difference between Azure Stack Hub & Azure Stack HCI
* Stack Hub 是讓你可以在 disconnected 的環境下有一整個像是 Azure Cloud 的環境還有 Azure service 可以用，基本上就是幫你 build 一個 PaaS 的 data center。
* Stack HCI 是一套 virtualization 的 solution，跟 vmware 什麼的比較像，只是說他有 AKS extension 讓你可以跑 on-prem k8s。不過另外你可以透過 Azure Arc，把 local 的 resource 加到 Azure 上，讓你原本在 cloud 的 services 可以跑到 local 來，變成類似 hybrid cloud 那樣。
* Stack Hub 買起來 minimum 就要 4台 server + network switch，Stack HCI 只要 2台 server 然後 network back-to-back 連就好
![azure stack](https://docs.microsoft.com/en-us/azure-stack/operator/media/compare-azure-azure-stack/azure-family-updated.png?view=azs-2108)

### Azure Arc
雖然跟我們要做的事情（在 local 跑 managed k8s）沒有什麼直接的關係，他其實是一個 hybrid cloud solution，不過因為很常被拿來跟 Azure Stack HCI 放在一起講，所以這裡寫一下，以免搞混
* Azure Arc 讓你可以把 on-prem 或是其他 cloud (ex aws, gcp...) 的 resource 加到 Azure (Arc-enabled server, Arc-enabled k8s)，讓你可以在上面跑 Azure services
* 所以你可以用 Stack HCI 開 VM 或 AKS，然後把他們加到 Azure Arc，這樣你就有 hybrid cloud 了
* 所以 Arc-enabled k8s 是讓你把不是 AKS 的 k8s 加到 Azure 裡面來使用管理，而不是幫你在 on-prem 跑 k8s，跟前面提的 k8s on Azure Stack HCI/Hub 是完全兩回事
* Arc–enabled services（ex Azure App Service / Azure Functions / Azure Logic Apps ...） 是你把你的 resource 加到 Azure Arc 後，你可以讓這些 service 跑到你加近來的 resource 上。

# AWS
## AWS Outposts
* AWS Outposts 可以買一整個 42U rack 或單台 1U/2U server 放在自己的 data center，然後你就可以在 local 跑 EC2, ECS, EKS, S3, ALB... 等 AWS services（只少要買到 rack 才能跑 EKS）
* Outposts 只在[某些 AWS Region 的國家&地區](https://aws.amazon.com/outposts/rack/faqs/)提供服務
* Outposts 機器基本上必須要[跟 AWS 有穩定且持續的連線](https://aws.amazon.com/outposts/rack/faqs/)，因為 control plane 都還是在 AWS 上，Outposts 只是讓你把 computing 和 data 放到 local
* 在 AWS cloud 的那 console 裡面有一個 tab 是 Outposts，幾本管理介面是 embedded 在原本的 Web GUI 裡。開 resource 基本上是以 subnet 作區分，Outposts 的 resource 會在一個 subnet 裡。
* 用 Outposts 跑 EKS 基本上是開一個一般的 EKS，然後 nodeGroup 的 subnet 設成 Outposts 的 subnet
* 當 Outposts 的跟 AWS 的連線斷掉時，local k8s node 可以繼續跑，但沒辦法做任何改變包括 scaling, failover 等等（基本上就是 k8s master 被拔開）

## ECS/EKS Anywhere
* open sourced
* 可以把跟 AWS EKS 一樣的 distro 裝在任何地方，self-managed EKS，不需要網路連接
* AWS 賣的是 Amazon EKS Anywhere Support，一年 24,000 USD，三年 54,000 USD
* 主要是用 eksctl + yaml 維護，node scaling 基本上就是改 yaml apply 就好，所以還可以搭配 flux 做 gitops
* 目前支援跑在 vsphere上，聽說 2022（不就是今年嗎，東西勒 XD）可以支援 bare metal

## Appendix
以下的 solution 跟我們比較沒有關係，但是都被放在 AWS hybrid solution，所以就稍微列一下
### AWS Wavelength
* 基本上是特別的 zones，像是 `us-east-1` 是一般機器跑在 AWS datacenter 的 zone，而 `us-east-1-wl1-bos-wlz-1` 就是一個機器跑在 Verizon 5G network IDC 的 zone
### AWS Local Zones
* 也是特別的 zones，提供更小的 latency
### AWS Snow
* AWS 提供可以讓你在完全沒有網路的情況做一些邊緣運算，然後在把 data physical 的送回 AWS datacenter
* 幾本上只能跑比較簡單的東西，像是開 EC2，沒有直接提供 EKS 跑法
* 有 GUI 叫做 AWS OpsHub
* 有三種 solution，都是特規機器，甚至還有卡車....
	* AWS Snowcone
	![](https://d1.awsstatic.com/cloud-storage/Storage/AWS-Snowcone.650f397305c8b7e9891b72d6b6dd490b0985e735.png)
	* AWS Snowball
	![](https://docs.aws.amazon.com/zh_tw/snowball/latest/developer-guide/images/Snowball-Edge-Image.png)
	* AWS Snowmobile
![](https://www.pcmarket.com.hk/wp-content/uploads/2016/12/DSC08286JPG.jpg)

## EKS Anywhere
