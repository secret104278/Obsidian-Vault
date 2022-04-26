# Blob Storage
[Supported Features Table](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-feature-support-in-storage-accounts)

## [Redunduency](https://docs.microsoft.com/en-us/azure/storage/common/storage-redundancy) 
- Locally redundant storage (LRS)
	- **synchronously** replica **within a single physical location** in the primary region
	- server rack and drive failures
- Zone-redundant storage (ZRS)
	- **synchronously** replica **across three zones** in the primary region
	- data center disaster
	- failover 需要一點時間，app 最好要自己做 exponential back-off retry

geo-redundant = **asynchronously** replica to **a single physical location** in secondary region with LRS

asynchronously 複製的時間差約為 15min.
- Geo redundant storage (GRS)
	- LRS + geo-redundant
- Geo Zone-redundant storage (GZRS)
	- ZRS + geo-redundant
app 要在 primary failure 才能 r/w secondary 的資料，不過可以另外 enable RA-GRS 或 RA-GZRS，這樣可以事前測試 app 是不是能成功 failover

## [Three type of blob](https://docs.microsoft.com/en-us/rest/api/storageservices/understanding-block-blobs--append-blobs--and-page-blobs)
- block blob
	- 實際上儲存的單位是 block，blob 實際上是 list of blocks，紀錄 block id list 。
	- 一個 block blob 最多可以包涵 50,000 blocks，每個 block 的大小不一定要一樣，block 最大約為 4000 MiB，所以單一 blob 最大約為 190.7 TiB。
	- block 一定會屬於某個 blob ，分為 committed 或 uncommitted
	- client 可以先平行 upload blocks 最後再 upload block id list 把更動 commit 進去
	- client 可以 replace/insert/delete 某個單獨的 block，最後再一次 commit
	- block list id 裡的 id 可以重複
	- 每個 block 可以有 MD5 checksum
	- [optimized for large amounts of read-heavy data](https://docs.microsoft.com/en-us/azure/storage/blobs/network-file-system-protocol-support#data-stored-as-block-blobs)

	**實驗 blobfuse 1.4.3 的行為**
	- 只支援 block blob
	- file size > 64MiB 會被切成多個 blocks
	- block size default 為 16MiB，假如需要超過 blocks 數量上限 (50,000) 會自動調整
	- block id = `base64 (idx (從 0 開始，開頭 padding 0到長度為 6) + 長度 36 uuid)`  eg. 第二個 block (16M~32M) 為  `000000000001f5a8db95-6e53-44c2-b40c-d58565b24890`
	- `flush()` 時才 upload 整個 file
	- 即使只 modify 一小段還是會 upload 整個 file（照 block blob 的設計應該要可以只 upload 被改的那個 block 才對）
	- concurrently upload blocks，mount option 可以設定 `--max-concurrency`
	- upload request sequence example ![[Pasted image 20220424024054.png]]
	- block id list example
```xml
<?xml version="1.0" encoding="utf-8"?>
<BlockList><Uncommitted>MDAwMDAwMDAwMDAwODRhN2UxN2YtNjk2ZC00NmQ0LWIyNWQtM2QzZWQ1OTRiYmIw</Uncommitted><Uncommitted>MDAwMDAwMDAwMDAxZjVhOGRiOTUtNmU1My00NGMyLWI0MGMtZDU4NTY1YjI0ODkw</Uncommitted><Uncommitted>MDAwMDAwMDAwMDAyNGNiZjZjOGQtMDIxYS00OTZiLWFkMDQtMjMwM2QzZjBjZGMx</Uncommitted><Uncommitted>MDAwMDAwMDAwMDAzYmVhMjc1OTAtODI3Ny00YjAwLThmOWEtOGVhOGNiODYxNjQz</Uncommitted><Uncommitted>MDAwMDAwMDAwMDA0OTM5N2U1YTMtYWExMi00N2MwLWE4MmQtMTFiYWJhYzJlMmQz</Uncommitted>
</BlockList>
```

- page blob
	- 儲存單位是大小為 512 byte 的 page，適合 random read/write，每次 update 都是以 pages 為單位，並且會立刻 commit
	- blob 建立的時候要宣告 blob size，上限為 8 TB。之後可以再 resize
	- Azure VM 的 disk、Azure DB 就都是用 page blob

- append blob
	- 跟 block blob 有點類似，都是由 block 組成
	- 只能 append block 在 blob 尾端，不能 update / delete block
	- 不需要 user 處理 block id，只需要 call  `Append Block`  一隻 request
	- 每個 block 最大只有 4 MiB，比 block blob 小，一個 blob 最大只有 195 GiB

### Note
* 在一般的 blob， folder 也是一個 blob，只是有 `x-ms-meta-hdi_isfolder=true` 的 metadata。在 data lake，folder 則是一個特殊的 `resource=directory`
* 有現成的 tool 能 mount 成 filesystem 只有 blobfuse + block blob / data lake 或 nfs + data lake

## Tiering
- 三種 tier - Hot, Cool, Archive
- tiering 只支援 Standard account type，不支援 Premium
- 在 Archive 的檔案是不可 read/write 的
- 每個 tier 都有規定檔案最短要存在該 tier 多久，否則會收 early deletion fee
	- 例外：假如 default 是設 Cool，但是提早把檔案移到 Archive 的話不會收 early deletion fee
- Hot 移到 Cool / Archive，Cool 移到 Hot 都是立刻完成；Archive 移到 Cool / Hot （稱為 Rehydrating）最久可能要 15hr
- 可以手動 call api 改變 tier （[Set Blob Tier](https://docs.microsoft.com/en-us/rest/api/storageservices/set-blob-tier) / [Copy Blob](https://docs.microsoft.com/en-us/rest/api/storageservices/copy-blob)）或用 [Blob lifecycle management](https://docs.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)
	- Copy Blob 可以用在暫時把檔案從 cooler 複製到 hotter tier，雖然佔兩倍空間的錢，但不會被收 early deletion fee

## Azure Data Lake Storage Gen2
AZ data lake = blob storage + Enable hierarchical namespace

特點：
- Atomic directory manipulation
	
	一般 object storage 再 key 裡面用 "/" 來仿造出 FS 的樣貌，但對 moving, renaming or deleting directories 等事實上都是 traversal operation。
	
	Use case: 像 Hive, Spark 會先寫到 temp dir 在做 rename dir 之類的，有真的 directory operation support 的話就能 O(1) 完成。

- Familiar Interface Style

也就是說如果用不到 directory management 的話，基本上沒有什麼好處。
> Some workloads might not gain any benefit by enabling a hierarchical namespace. Examples include backups, image storage, and other applications where object organization is stored separately from the objects themselves (for example: in a separate database).
> - https://docs.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-namespace#deciding-whether-to-enable-a-hierarchical-namespace

### NFS v3.0 support
- for high throughput, high scale, read heavy workloads
- base on block blobs + data lake，但實際上 behavior 跟 blobfuse 不一樣
	- 從 nfs 寫入的大檔後， `get_block_list` 是空的
	- 從 blobfuse 寫入的檔案沒有辦法在 nfs mount 裡更改，推測 blobfuse 寫入是真的用一堆 block + block id list，但 nfs mount 是用別的機制
	- blobfuse random write/append 檔案會在 `open()` 時下載整個檔案到 local，然後最後 `flush()` 的時上傳整個檔案。nfs mount 就像是一般 nfs 只上傳或是下載 request 的片段。
- [目前只能用 VNet 去做 security 的限制](https://docs.microsoft.com/en-us/azure/storage/blobs/network-file-system-protocol-support#network-security)
- [create storage account 之後就沒辦法再改，不支援 geo redundancy](https://docs.microsoft.com/en-us/azure/storage/blobs/network-file-system-protocol-known-issues)

