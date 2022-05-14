Beam 是一個 high level 的 interface，其實際的功能都還是依賴於底層的 runner （如 spark 等等）
## Basic
### PCollection
* P- stands for parallel
* 代表 list of bounded/unbounded elements
* **NO** order guarantees

### PTransform
* 代表一個 function
* 基本：Map one-to-one，FlatMap: one-to-many，Combine: many-to-one，GroupByKey: group related elements

## Windowing
