# Azure Stack Hub Deep Dive
## Compute
### Availability
* A fault domain = 1 台 physical node
* A availability set = 3 fault domains = 3 台 physical node
* each instance in a virtual machine scale set is placed in a different fault domain
=> 一個 HA replica = 3 的 scale set 會需要至少 9 台 physical node
* GPU VMs 沒有 HA mode

### Resource
* ✅ overcommit CPU, ❌ doesn't overcommit MEM
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

## Storage
### Concept
- 整個系統 NVME, SATA/SAS SSD, HDD 3 選 2或1 （不能全 HDD），比較快的會變成 caching drives，比較慢的變 capacity drives
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-storage-infrastructure-overview/image1.png?view=azs-2108)
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-storage-infrastructure-overview/image2.png?view=azs-2108)
- All flash (ex NVME + SSD) 只有 write cache
- Hybrid (ex SSD + HDD) 有 read/write cache
![](https://docs.microsoft.com/en-us/azure-stack/operator/media/azure-stack-storage-infrastructure-overview/image3.svg?view=azs-2108)
- 一組 rack (scale unit) 會組成一個 Storage Spaces Direct Pool，pool 底下再分 volume，每加一台 physical node 就會多一個 VM Temp volumes 和一個 Object Store volumes
- 三種 volume
	- Infrastructure volumes: 給 Stack Hub 自己用的
	- VM Temp volumes: 每個 VMs 都會有 temporary disks
	- Object Store volumes: blobs, tables, queues, VM disks
- Three-way mirroring，data 會被 striping 存到三個不同機器上的不同 caching drives 上
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
