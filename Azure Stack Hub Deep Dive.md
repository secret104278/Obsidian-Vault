# Azure Stack Hub Deep Dive
## Open question
- physical-to-virtual cpu mapping?

## Datacenter integration
### Management
- Stack Hub 整體來說是一個滿封閉的系統，基本上一般的 maintain 都是透過 Portal 或是 API 等等，不太能直接 access physical 機器

### Connect Azure Stack Hub to Azure
- 可以完全不連
- Site-to-site VPN over IPsec，max BW =~ 100-200 Mbps
- Outbound NAT，只能單方向連線
- [ExpressRoute](https://docs.microsoft.com/en-us/azure-stack/operator/azure-stack-datacenter-integration?view=azs-2108#using-expressroute)
- [Hybrid connectivity options](https://docs.microsoft.com/en-us/azure-stack/operator/azure-stack-datacenter-integration?view=azs-2108#hybrid-connectivity-options)
- [External monitoring from Azure](https://docs.microsoft.com/en-us/azure-stack/operator/azure-stack-integrate-monitor?view=azs-2108)

### Identity provider & License (Billing) & Connection Mode
一開始選好就不能改
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-connection-models/azure-stack-scenarios.png?view=azs-2108)

### [DNS Naming](https://docs.microsoft.com/en-us/azure-stack/operator/azure-stack-integrate-dns?view=azs-2108)
需要決定這三個
- Region name (ex. saudiarabia1)
- External domain name (對外部的 DNS) zone，ex. evai.foxcoon.com)
- Private (internal) domain name (Stack Hub infrastructure 用的)

最後的 FQDN 由 Region name + External domain name 組成，例如
- https://portal.saudiarabia1.evai.foxcoon.com
- https://myapp.saudiarabia1.evai.foxcoon.com

![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-integrate-dns/integrate-dns-01.svg?view=azs-2108)
- Stack Hub 內部會有 Recursive Resolver（負責轉法 DNS query），和 Authoritative Resolver （回復負責的 zones 的 DNS query）
- 把 Recursive Resolver 跟外部 DNS 對接，提供 Stack Hub 內 VMs DNS 和外面的 public access

### Certificate
https://docs.microsoft.com/en-us/azure-stack/operator/azure-stack-pki-certs?view=azs-2108
- Stack Hub 需要 certificate for infrastructure SSL 和 public endpoint
- 可以用 public CA 或 enterprise internal CA（required when disconnected deployment），不能用 self-signed certificates。Stack Hub 要可以 access 到 CA 上的 Certificate Revocation List (CRL)
- [Generate CSR](https://docs.microsoft.com/en-us/azure-stack/operator/azure-stack-get-pki-certs?view=azs-2108&tabs=omit-cn&pivots=csr-type-new-deployment#generate-csrs-for-new-deployment-certificates)

### Network
https://docs.microsoft.com/en-us/azure-stack/operator/azure-stack-network?view=azs-2108
- 一個 scale unit (rack) 的 physical topology
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-network/physical-network.svg?view=azs-2108)
- 整個Stack Hub（包括 VMs 的 ip 等等） 的 topology ![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-network/networkdiagram.svg?view=azs-2108)
- Stack Hub 裡面會有 SDN，主要是要決定一組 public VIP sub 讓你可以從 Stack Hub 外面連進來。Stack Hub 本身會用掉其中 31 個 ip，一些像 portal 或 Azure services 也會需要，剩下的就是給 VM 開 public ip 用。
- note: 如果 Stack Hub 用 Site to Site 跟 on-prem 內網連的話，要[特別注意 NAT routing](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-network/pvip-2.png?view=azs-2108)
	例如有 on-prem 內網 `10.100.*.*` ，stack hub 內網 `10.0.*.*` ，兩個內網有 Site-to-Site VPN 連起來。這時 on-prem client `10.100.0.10` 打 VM 的 public ip `137.72.10.40` 會從綠色的 gateway 進去（沒有過任何 NAT），然後 VM 回傳時會直接用 stack hub private ip `10.0.0.4` 經過 s2s vpn gateway 回去，然後最後回到 on-prem client 的兩個 ip 不一樣（`137.72.10.40` 和 `10.0.0.4`），連線就壞掉了。
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-network/pvip-1-expanded.png?view=azs-2108#lightbox)
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-network/pvip-2.png?view=azs-2108)


### Backup and disaster recovery
https://docs.microsoft.com/en-us/azure-stack/operator/azure-stack-backup-back-up-azure-stack?view=azs-2108
- Stack Hub 有選項可以備份 Stack Hub 本身的 Infrastructure 設定，例如：
	- Role-based access permissions and roles.
	- Original plans and offers.
	- Previously defined compute, storage, and network quotas.
	- Key Vault secrets.
- 但其他像是 VMs 裡面的資料，或是 blob storage 等等的，user 要另外自己設定

## Computing in Stack Hub
### Availability
* A fault domain = 1 台 physical node
* A availability set = 3 fault domains = 3 台 physical node
* each instance in a virtual machine scale set is placed in a different fault domain
	- => 一個 HA replica = 3 的 scale set 會需要至少 9 台 physical node
* GPU VMs 沒有 HA mode

### Resource
* ✅ overcommit CPU, ❌ doesn't overcommit MEM （how??）
* placement algorithms 不管 CPU overprovisioning ratio，所以每台機器可能不一樣
* 一台 physical 機器最多 60 vm，一整個 Azure Stack Hub 最多 700 vm
* 開 vm 的速度是：一次 40 ~ 30 台，每次間隔 5 mins
* Available memory for VM placement = total host memory - resiliency reserve - memory used by running tenant VMs - Azure Stack Hub Infrastructure Overhead
	* Resiliency reserve = `N * R + (N-2)* V + min(H-R, sum(memoryofHAVMs))`
		- N = number of physical servers
		- R = OS overhead, 15% of physical node memory
		- V = Largest HA VM memory (最小是 12 GB 給 Stack Hub 本身的 VM。Stack Hub 支援最大的 vm memory 是 112 GB。不包括 GPU VM  因為是 none-HA)
		- H = Size of single server memory
		- memoryofHAVMs = sum of the memory of all the HA VMs in the scale unit. This includes 142 GB of HA infrastructure + memory of HA tenant VMs
	* Azure Stack Hub Infrastructure overhead = 約 31 VMs，佔 268 GB + (4 GB x # of physical nodes) + 146 vCPU.
	
	=> 盡量 Reduce the size of the largest VM

## Storage in Stack Hub
### Concept
- 整個系統 NVME, SATA/SAS SSD, HDD 3 選 2或1 （不能全 HDD），比較快的會變成 caching drives，比較慢的變 capacity drives。
	- 開 blob 或 disk 的時候選 premium type 是不會影響資料是存在什麼東西上面的。
	- 不同 size 的 VM 的 IOPS 都有固定的限制，避免 noise neighbor
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-storage-infrastructure-overview/image1.png?view=azs-2108)
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-storage-infrastructure-overview/image2.png?view=azs-2108)
- All flash (ex NVME + SSD) 只有 write cache
- Hybrid (ex SSD + HDD) 有 read/write cache
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-storage-infrastructure-overview/image3.svg?view=azs-2108)


- 一組 rack (scale unit) 會組成一個 Storage Spaces Direct Pool，pool 底下再分 volume，每加一台 physical node 就會多一個 VM Temp volumes 和一個 Object Store volumes
- 三種 volume
	- Infrastructure volumes: 給 Stack Hub 自己用的
	- VM Temp volumes: 每個 VMs 都會有 temporary disks，是直接從 host machine 掛上去的
	- Object Store volumes: blobs, tables, queues, VM disks
- Three-way mirroring，data 會被 striping 然後 duplicated 存到三個不同機器上的不同 caching drives 上
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-storage-infrastructure-overview/image5.png?view=azs-2108)

### Resource
- 每台 physical node 需要至少 340 GB 做 boot/local
- Azure Stack Hub infrastructure 需要 3.5 TB
- Virtual disk capacity is calculated and allocated in a way as to leave one capacity device's amount of data capacity unallocated in the pool. This is the equivalent of one capacity drive per server（什麼意思？？）
- VM Temp volumes 大小算法（Server 是指 physical node，一台 physical unit 就會有一個TempVirtualDisk，我猜他應該就是指 VM Temp volumes？）
```
DesiredTempStoragePerServer = PhysicalMemory * 0.65 * 8
  TempStoragePerSolution = DesiredTempStoragePerServer * NumberOfServers
  PercentOfTotalCapacity = TempStoragePerSolution / TotalAvailableCapacity
  If (PercentOfTotalCapacity <= 0.1)
      TempVirtualDiskSize = DesiredTempStoragePerServer
  Else
      TempVirtualDiskSize = (TotalAvailableCapacity * 0.1) / NumberOfServers
```
