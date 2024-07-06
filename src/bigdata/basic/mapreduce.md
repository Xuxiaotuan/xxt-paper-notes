### [MapReduce: Simplified Data Processing on Large Clusters](../../assets/pdfs/MapReduce-Simplified-Data-Processing-on-Large-Clusters.pdf)


> OSDI'04: Sixth Symposium on Operating System Design and Implementation, San Francisco, CA (2004), pp. 137-150
>
> https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf# mapreduce


在处理 Distributed Grep，Inverted Index、Distributed Sort 等问题时，虽然数据本身需要执行的转换非常简单，但在高度分布式、可扩展和容错的环境中执行这些任务却又不那么简单，MapReduce 通过隐藏所有分布式系统的复杂性，为用户提供了一个分布式计算框架，用户只需提供用于将 key/value pair 处理生成一组 intermedia key/value pairs 的 map 函数，和一个将同一个键对应的所有 intermedia key/value pairs 做合并操作的 reduce 函数，就可以将程序并行地运行在计算机集群上。

MapReduce 的执行过程如下:

![MapReduce](./../../assets/images/bigdata/basic/mapreduce/mapreduce.png)

1. **输入分割**：用户程序内的MapReduce库首先将输入文件切割成M块，每块大小一般为16至64兆字节（MB），具体由用户通过可选参数设定。之后，在机器集群上启动程序的多个副本。

2. **任务分配与主从架构**：这些副本中，有一个作为特殊角色——主节点，负责调度工作；其余副本作为工作节点，接受主节点指派的任务。主节点需分配M个映射任务和R个归约任务。它会选择空闲的工作节点，分别给予映射或归约任务。

3. **映射阶段**：获得映射任务的工作节点读取对应输入块的内容，从中解析出键值对，并将每对键值传递给用户自定义的Map函数处理。Map函数产生的中间键值对先在内存中缓存。

4. **中间结果存储**：缓存的键值对会周期性地写入本地磁盘，并通过分区函数划分成R个区域。这些数据在磁盘上的位置信息反馈给主节点，主节点再转发给归约工作节点。

5. **数据传输与排序**：归约工作节点接收到位置信息后，利用远程过程调用读取各映射工作节点上的中间数据。数据全部读取完毕后，按中间键进行排序，确保相同键的值聚合。若中间数据量超出内存容量，则采用外部排序。

6. **归约处理**：归约工作节点遍历排序后的中间数据，对每个唯一中间键，将该键及关联的中间值集传递给用户定义的Reduce函数处理。Reduce函数的输出追加到相应归约分区的最终输出文件中。

7. **任务完成与程序结束**：所有映射和归约任务执行完毕后，主节点激活用户程序，此时用户程序中的MapReduce调用结束，控制权返回用户代码。

一个 MapReduce 任务可以分为三个阶段:

- map phase: 在 map worker 上，处理后的中间数据根据默认或用户提供的 partitioning function 将数据保存为 R 个本地文件，并将文件位置上报给 master
- shuffle phase: 在 reduce worker 上，根据从 master 上获取的文件位置，从各个 map worker 上读取所需的文件
- reduce phase: 在 reduce worker 上将读取文件中 intermedia key/value pairs 进行处理的过程

Fault Tolerance
当 master 失败时，整个任务重做；当 map worker 失败时，即使它已经完成，也需要重做，因为中间数据文件是写在本地的；当 reduce worker 失败时，如果任务未完成，需要成重新调度其他节点完成对应的 reduce 任务，如果任务已经完成，则不需要重做，因为 reduce 的结果保存在 GFS。

优化
- Locality: 由于 input file 保存在 GFS 上，MapReduce 可以根据文件存储的位置，将 map worker 调度到数据分片所在的节点上以减少网络开销
- Combiner: 当 reduce 函数满足交换律和结合律特性时，可以将 reduce 的工作在 map 阶段提前执行
- Backup Tasks: 将一定比例的长尾任务重新调度，可以减少任务的整体执行时间
Apache Hadoop 是 MapReduce 的开源实现，2014 年 Google 提出了 MapReduce 的替代模型 Cloud Dataflow，该模型支持流批一体，具有更好的性能及扩展性，对标的开源产品为 Apache Flink。