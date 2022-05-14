## Some history
- google's 3 paper about big data: [Google File System (2003)](https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf), [MapReduce (2004)](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf), and [Bigtable (2006)](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)
	- GFS - fault-tolerant and distributed filesystem across many commodity hardware servers in a cluster farm
	- Bigtable - scalable storage of structured data across GFS
	- MapReduce - parallel programming paradigm, based on functional programming, for large-scale processing of data distributed over GFS and Bigtable
- yahoo 和社群在 2005 基於 google 的 paper 實作 Hadoop framework: Hadoop Common, MapReduce, HDFS, and Apache Hadoop YARN
- AMPLab (RISELab) 在 2009 開始開發 spark 並在 2014 release 1.0，主要是為了要解決 Hadoop inefficient (or intractable) for interactive or iterative computing jobs and a complex framework to learn 的問題。後來貢獻給 apache 並創立了 Databricks
- Hadoop 的一些缺點：
	- hard to manage and administer
	- MapReduce API was verbose and required a lot of boilerplate setup code, with brittle fault tolerance
	- High disk I/O cost from each MR pair’s intermediate computed result is written to the local disk for the subsequent stage of its operation
	- fell short for combining other workloads such as machine learning, streaming, or interactive SQL-like queries
- 後來 Hadoop 生態系也出現了 Apache Hive, Apache Storm, Apache Impala, Apache Giraph, Apache Drill, Apache Mahout 等等 projects，試圖補足上述的問題

## Advantage of Spark
- Speed
	- Hadoop 和早期 MR 的硬體網路 performance 跟現在已經差很多了，很多當時的 trade-off 例如為了避免網路 bottleneck 把 storage 跟 computing 綁在同一台機器上等等的已經不適用了
	- Spark 的 query computations 是設計成 DAG，而這個 DAG scheduler 和 query optimizer 能夠有效得把 task 平行的跑在 cluster 上
	- Tungsten - spark 的 physical execution engine，uses whole-stage code generation to generate compact code for execution
	- intermediate result 盡量放在 memory，減少 disk io 開銷
- Ease of Use
- Modularity
	- Spark SQL, Spark Structured Streaming, Spark MLlib, and GraphX 等等的應用都在同一套 engine 上
- [Extensibility](https://spark.apache.org/third-party-projects.html)
	- Spark 可以對接各種 storage - Hadoop, Apache Cassandra, Apache HBase, MongoDB, Apache Hive, RDBMSs, and more
	- Spark 的 DataFrameReaders 和 DataFrame Writers 也可以對接 Apache Kafka, Kinesis, Azure Storage, and Amazon S3 等等

## RDDs / DataFrames / DataSets
A Tale of Three Apache Spark APIs: RDDs vs DataFrames and Datasets [blog](https://databricks.com/blog/2016/07/14/a-tale-of-three-apache-spark-apis-rdds-dataframes-and-datasets.html?utm_source=twitter&utm_medium=cpc&utm_content=blog&utm_campaign=21726252&utm_offer=a-tale-of-three-apache-spark-apis-rdds-dataframes-and-datasets) [talk](https://youtu.be/Ofk7G3GD9jk)

### RDD
Spark 最基本的 data 單位，即使在後來的 DataFrames / DataSets 也沒有消失，他只是存在更底層而已。
*  distributed data abstraction
	* partition
* Resilient & Immutable
	* RDD 會紀錄他是從哪個 parent RDD 經過什麼 transform 得到的。也就是當 program crash 的之後，還是可以重新算出來。
* compile-time type-safe
	* RDD 裡面的資料是有 typing 的 (int, str, binary ....)
* unstructured / structured data
* Lazy
	* transform operations 直到真的呼叫 action operations 都不會被執行

#### When to use RDD
* low-level control
* unstructured data （指的就是沒有 schema / columnar format 的）
* 想直接寫 functional programming 而不是 DSL （domain specific language，像是 sql 那樣）
* sacrifice optimization

#### Problem of RDD
RDD 因為非常 low-level，所以是 user 直接告訴 spark 要去做什麼事情，而不是 user 想要做什麼。導致說 spark 就只能說什麼做什麼，沒有機會做 optimization。另外有時候 user 以為怎樣寫可以 optimize，但往往容易有誤區，導致最後變得更 inefficient。
總結來說就是，問題就是 user 直接操作 RDD 對 spark 來說是 opaque computation + opaque data

### DataFrame
DataFrame 將資料變成 named columns 也就是 structured （這也是為什麼後來改用 DataFrame 的 streaming 會叫做 structured streaming 的原因），並提供 higher-level 的 DSL。而 spark 基本上就能根據 schema 和 high-level operation 去做 optimization  ([Catalyst optimizer](https://databricks.com/glossary/catalyst-optimizer))
![[Pasted image 20220513143546.png]]
![[Pasted image 20220513143725.png]]
### DataSet
DataFrame 因為沒辦法有 compile-time type-safe，所以 spark 針對 strongly typed language 提供 DataSet Abstraction。但這基本上是在 SDK 上的差異，實際背後還是跟 DataFrame 的邏輯一樣，也因此只有 scalar 和 java 有這東西， python 和 R 是沒有的。
![[Pasted image 20220513143605.png]]

![[Pasted image 20220513143658.png]]