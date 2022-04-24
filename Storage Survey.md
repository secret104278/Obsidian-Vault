# Storage

## Common Storage Systems
* NAS
* [Ceph](https://docs.ceph.com/en/pacific/)
* [Lustre](https://www.lustre.org/)
    感覺比 ceph 更主流的專攻 FS storage。近來有推出一些針對 small file 優化的 feature。
    很多 feature 都是 [DDN](https://www.ddn.com/) 貢獻給社群的，cloud 上好像可以直接 bootstrap 他們的系統起來用用看 [GCP](https://console.cloud.google.com/marketplace/vm/config/ddnstorage/exascaler-cloud?project=lustre-347008) [AWS](https://aws.amazon.com/marketplace/pp/prodview-tdfa4jyemfcdc) [Azure](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/ddn-whamcloud-5345716.exascaler_cloud_app?tab=overview)
* [BeeGFS](https://www.beegfs.io/c/)
* [GlusterFS](https://www.gluster.org/)
* [DAOS](https://docs.daos.io/v2.0/)
    基於 intel optane 的 storage class memory PMEM/NVMe 的超高速 object storage，不打算支援 block device。
    實際上的用途類似於 GekkoFS 這種 burst buffer / performance tier
    有開發跟 lustre 的 [integration](https://www.opensfs.org/wp-content/uploads/2019/07/LUG2019-Cross_Tier_Unified_Namespace-Chaarawi.pdf)，似乎是可以再 lustre mount 裡面的某個 dir 瀏覽 daos 的內容。
* [SeaweedFS](https://github.com/chrislusf/seaweedfs)
    某中國人跟據 facebook paper 實作的 key - large value 的 storage。
* [Minio](https://min.io/)
    不好 scale 的 s3 storage
* [GekkoFS](https://github.com/NGIOproject/old_GekkoFS_old)
     burst buffer storage，不太能算是一個 system。
     用 LD_PRELOAD 的方式 hook io 在 clients 端做 buffer，再後續寫回其他 storage system

## filesystem as local cache
s3fuse(AWS), blobfuse(Azure) 和 gcsfuse(GCP) 都有相關的 local fs as cache 的機制。

## Problem with ceph
### Imbalance distribution over disks
ceph 的 file -> objects 是 `object id = <ino in hex>.<object in sequence>`。object 則是由 CRUSH algorithm 計算 object -> pg (placement group) -> osd。之前因為 pg 數量沒有隨 cluster scale up 被正確設定時，導致分佈不均。
    
lustre 的 file -> objects 關係是存在 MDT inode 裡名為 layout EA 的 extended attribut。lustre 的 OST 會不平均通常是因為錯誤的 striping 設定，但 lustre 還可以透過 MDS 手動停止繼續在某個 OST create objects。
https://doc.lustre.org/lustre_manual.xhtml#Fig1.3_LayoutEAonMDT
https://doc.lustre.org/lustre_manual.xhtml#handling_full_ost
    
### Slow single user performance at small iosize (striping configuration)
ceph 的 strip 是由 stripe unit size + stripe count + object size 決定的。
https://docs.ceph.com/en/latest/dev/file-striping/

lustre 的一個 file 最多只能由 2000 個 objects。

striping 在這種複雜的 distributed storage 比較適用於多 thread 讀寫大檔案。試想假如是小檔案配小 iosize 時，一個 io 操作後面要 trigger 很多 network requests 反而可能導致更多 overhead。另外以 server 的角度來說，反而每個 disk 的 io pattern 會更 random，變成每個 client 會互相 competet
感覺 distributed storage 要單個 client latency 小的解法應該是要有 all-flash tier。lustre 有 built-in Persistent Client Cache (PCC) 的機制，可以在 client node 掛 nvme 或 ssd 接到 lustre，對 user 來說是 transparent 的 caching。（另外 lustre 在實作 PCC 的時候還有實作可以直接查 file heat 的指令）
https://doc.lustre.org/lustre_manual.xhtml#file_striping.considerations
https://doc.lustre.org/lustre_manual.xhtml#pcc
    
### Busy and memory saturated metadata server
 ceph 的client 每打開一個檔案，metadata server 都會需要 allocate 一些 memory 做管理（doc 裡面叫這個 object 爲 capability）。這個的用途類似分散式鎖，當一個 client 對某個檔案寫入的時候，其他 client 的 local cache 的 capability 會需要被剝奪以防止 consistent 的問題。這些事情會透過 metadata server 管理，所以大量的檔案或是 directory traversal 會對 metadata 造成巨大的影響

lustre 的 Lustre distributed lock manager (LDLM) 由 MDT 和 OST 共同負責，可以稍微分散壓力
* MDT LDLM manages locks on inode permissions and pathnames
* OST LDLM manages locks on file stripes stored thereon

lustre 也有提供 lockless io 的 tuning
https://doc.lustre.org/lustre_manual.xhtml#tuning_lockless_IO

另外 ceph 的 multi-mds 好像是去年才 remove experimental tag，但 lustre 從以前就有 multiple MDT （也是 sharding 的機制）

### No built-in backup tools
ceph 只有 snapshot功能

lustre 有提供 lustre_rsync 可以做 diff backup，因為 lustre 本身就有 audit log 功能，所以他可以知道哪些檔案有被改過。
https://doc.lustre.org/lustre_manual.xhtml#lustre_changelogs
https://doc.lustre.org/lustre_manual.xhtml#backupandrestore

lustre 也有提供 snapshot，前提時 server target 必須都是 ZFS
https://doc.lustre.org/lustre_manual.xhtml#zfssnapshots

### No multi-tiering solution
ceph 其實也是有 cache tier 的實作，但是 document 就直接寫說不 stable...

lustre 有 Hierarchical Storage Management (HSM) 的機制，可以把 data async 的 copy 到更便宜的 archives pool。另外也有 Persistent Client Cache (PCC) 的機制。

https://doc.lustre.org/lustre_manual.xhtml#lustrehsm
https://doc.lustre.org/lustre_manual.xhtml#pcc

### Client kernel die

lustre 和 ceph 的 client 都是 kernel-space 的，所以面不了一些這方面的問題，只能看誰實作得比較穩定。

### Failed to schedule data scrubbing


