---
title: Graphx 源码剖析-图的生成
date: 2016-10-31 23:34:52
categories: spark相关
tags: 
- spark
- graphx
---
Graphx的实现代码并不多，这得益于Spark RDD niubility的设计。众所周知，在分布式上做图计算需要考虑点、边的切割。而RDD本身是一个分布式的数据集，所以，做Graphx只需要把边和点用RDD表示出来就可以了。本文就是从这个角度来分析Graphx的运作基本原理（本文基于Spark2.0）。
<!--more-->
# 分布式图的切割方式
在单机上图很好表示，在分布式环境下，就涉及到一个问题：图如何切分，以及切分之后的不同子图如何保持彼此的联系构成一个完整的图。图的切分方式有两种：点切分和边切分。在Graphx中，采用点切分。

在GraphX中，`Graph`类除了表示点的`VertexRDD`和表示边的`EdgeRDD`外，还有一个将点的属性和边的属性都包含在内的`RDD[EdgeTriplet]`。
方便起见，我们先从`GraphLoader`中来看看如何从一个用边来描述图的文件中如何构建`Graph`的。
```
def edgeListFile(
      sc: SparkContext,
      path: String,
      canonicalOrientation: Boolean = false,
      numEdgePartitions: Int = -1,
      edgeStorageLevel: StorageLevel = StorageLevel.MEMORY_ONLY,
      vertexStorageLevel: StorageLevel = StorageLevel.MEMORY_ONLY)
    : Graph[Int, Int] =
  {

    // Parse the edge data table directly into edge partitions
    val lines = ... ...
    val edges = lines.mapPartitionsWithIndex { (pid, iter) =>
      ... ...
      Iterator((pid, builder.toEdgePartition))
    }.persist(edgeStorageLevel).setName("GraphLoader.edgeListFile - edges (%s)".format(path))
    edges.count()

    GraphImpl.fromEdgePartitions(edges, defaultVertexAttr = 1, edgeStorageLevel = edgeStorageLevel,
      vertexStorageLevel = vertexStorageLevel)
  } // end of edgeListFile
```
从上面精简的代码中可以看出来，先得到`lines`一个表示边的RDD（这里所谓的边依旧是文本描述的）,然后再经过一系列的转换来生成Graph。
# EdgeRDD
`GraphImpl.fromEdgePartitions`中传入的第一个参数`edges`为`EdgeRDD`的`EdgePartition`。先来看看`EdgePartition`究竟为何物。
```
class EdgePartition[
    @specialized(Char, Int, Boolean, Byte, Long, Float, Double) ED: ClassTag, VD: ClassTag](
    localSrcIds: Array[Int],
    localDstIds: Array[Int],
    data: Array[ED],
    index: GraphXPrimitiveKeyOpenHashMap[VertexId, Int],
    global2local: GraphXPrimitiveKeyOpenHashMap[VertexId, Int],
    local2global: Array[VertexId],
    vertexAttrs: Array[VD],
    activeSet: Option[VertexSet])
  extends Serializable {
```
其中：
`localSrcIds` 为本地边的源点的本地编号。
`localDstIds` 为本地边的目的点的本地编号，与`localSrcIds`一一对应成边的两个点。
`data` 为边的属性值。
`index` 为本地边的源点全局ID到localSrcIds中下标的映射。
`global2local` 为点的全局ID到本地ID的映射。
`local2global` 是一个Vector，依次存储了本地出现的点，包括跨节点的点。
通过这样的方式做到了点切割。
有了`EdgePartition`之后，再通过得到`EdgeRDD`就容易了。
# VertexRDD
现在看`fromEdgePartitions`
```
  def fromEdgePartitions[VD: ClassTag, ED: ClassTag](
      edgePartitions: RDD[(PartitionID, EdgePartition[ED, VD])],
      defaultVertexAttr: VD,
      edgeStorageLevel: StorageLevel,
      vertexStorageLevel: StorageLevel): GraphImpl[VD, ED] = {
    fromEdgeRDD(EdgeRDD.fromEdgePartitions(edgePartitions), defaultVertexAttr, edgeStorageLevel,
      vertexStorageLevel)
  }
```
`fromEdgePartitions` 中调用了 `fromEdgeRDD`
```
  private def fromEdgeRDD[VD: ClassTag, ED: ClassTag](
      edges: EdgeRDDImpl[ED, VD],
      defaultVertexAttr: VD,
      edgeStorageLevel: StorageLevel,
      vertexStorageLevel: StorageLevel): GraphImpl[VD, ED] = {
    val edgesCached = edges.withTargetStorageLevel(edgeStorageLevel).cache()
    val vertices =
      VertexRDD.fromEdges(edgesCached, edgesCached.partitions.length, defaultVertexAttr)
      .withTargetStorageLevel(vertexStorageLevel)
    fromExistingRDDs(vertices, edgesCached)
  }
```
可见，`VertexRDD`是由`EdgeRDD`生成的。接下来讲解怎么从`EdgeRDD`生成`VertexRDD`。
```
def fromEdges[VD: ClassTag](
      edges: EdgeRDD[_], numPartitions: Int, defaultVal: VD): VertexRDD[VD] = {
    val routingTables = createRoutingTables(edges, new HashPartitioner(numPartitions))
    val vertexPartitions = routingTables.mapPartitions({ routingTableIter =>
      val routingTable =
        if (routingTableIter.hasNext) routingTableIter.next() else RoutingTablePartition.empty
      Iterator(ShippableVertexPartition(Iterator.empty, routingTable, defaultVal))
    }, preservesPartitioning = true)
    new VertexRDDImpl(vertexPartitions)
  }

  private[graphx] def createRoutingTables(
      edges: EdgeRDD[_], vertexPartitioner: Partitioner): RDD[RoutingTablePartition] = {
    // Determine which vertices each edge partition needs by creating a mapping from vid to pid.
    val vid2pid = edges.partitionsRDD.mapPartitions(_.flatMap(
      Function.tupled(RoutingTablePartition.edgePartitionToMsgs)))
      .setName("VertexRDD.createRoutingTables - vid2pid (aggregation)")

    val numEdgePartitions = edges.partitions.length
    vid2pid.partitionBy(vertexPartitioner).mapPartitions(
      iter => Iterator(RoutingTablePartition.fromMsgs(numEdgePartitions, iter)),
      preservesPartitioning = true)
  }
```
从代码中可以看到先创建了一个路由表，这个路由表的本质依旧是RDD，然后通过路由表的转得到`RDD[ShippableVertexPartition]`，最后再构造出`VertexRDD`。先讲解一下路由表，每一条边都有两个点，一个源点，一个终点。在构造路由表时，源点标记位或1，目标点标记位或2，并结合边的partitionID编码成一个Int（高2位表示源点终点，低30位表示边的partitionID）。再根据这个编码的Int反解出`ShippableVertexPartition`。值得注意的是，在`createRoutingTables`中，反解生成`ShippableVertexPartition`过程中根据点的id hash值partition了一次，这样，相同的点都在一个分区了。有意思的地方来了：我以为这样之后就会把点和这个点的镜像合成一个，然而实际上并没有。点和边是相互关联的，通过边生成点，通过点能找到边，如果合并了点和点的镜像，那也找不到某些边了。`ShippableVertexPartition`依旧以边的区分为标准，并记录了点的属性值，源点、终点信息，这样边和边的点，都在一个分区上。
最终，通过`new VertexRDDImpl(vertexPartitions)`生成`VertexRDD`。
# Graph
```
 def fromExistingRDDs[VD: ClassTag, ED: ClassTag](
      vertices: VertexRDD[VD],
      edges: EdgeRDD[ED]): GraphImpl[VD, ED] = {
    new GraphImpl(vertices, new ReplicatedVertexView(edges.asInstanceOf[EdgeRDDImpl[ED, VD]]))
  }
```
在`fromExistingRDDs`调用`new GraphImpl(vertices, new ReplicatedVertexView(edges.asInstanceOf[EdgeRDDImpl[ED, VD]]))`来生成图。
```
class ReplicatedVertexView[VD: ClassTag, ED: ClassTag](
    var edges: EdgeRDDImpl[ED, VD],
    var hasSrcId: Boolean = false,
    var hasDstId: Boolean = false)
```
`ReplicatedVertexView`是边和图的视图，当点的属性发生改变时，将改变传输到对应的边。