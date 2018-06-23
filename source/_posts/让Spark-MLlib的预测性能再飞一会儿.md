---
title: 让Spark MLlib的预测性能再飞一会儿
date: 2018-05-02 17:14:21
categories: spark相关
tags:
- spark
- MLlib
- 调优
---
# 背景介绍
我们的系统有一小部分机器学习模型识别需求，因为种种原因，最终选用了Spark MLlib来进行训练和预测。MLlib的Pipeline设计很好地契合了一个机器学习流水线，在模型训练和效果验证阶段，pipeline可以简化开发流程，然而在预测阶段，MLlib pipeline的表现有点差强人意。

<!--more-->
# 问题描述
某个模型的输入为一个字符串，假设长度为N，在我们的场景里面这个N一般不会大于10。特征也很简单，对于每一个输入，可以在O(N)的时间计算出特征向量，分类器选用的是随机森林。
对于这样的预测任务，直观上感觉应该非常快，初步估计10ms以内出结果。但是通MLlib pipeline的transform预测结果预测时，性能在86ms左右(2000条query平均响应时间)。而且，query和query之间在输入相同的情况下，也存在响应时间波动的问题。

# 预测性能优化
先说说响应时间波动的问题，每一条query的输入都是一样的，也就排除了特征加工时的计算量波动的问题，因为整个计算中消耗内存极少，且测试时内存足够，因为也排除gc导致预测性能抖动的问题。那么剩下的只有Spark了，Spark可能在做某些事情导致了预测性能抖动。通过查看log信息，可以印证这个观点。
![log](https://upload-images.jianshu.io/upload_images/2838375-267b2ad4c04beade.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从日志中截取了一小段，里面有大量的清理broadcast变量信息。这也为后续性能优化提供了一个方向。（下面会有部分MLlib源码，源码基于Spark2.3）

在MLlib中，是调用PipelineModel的transform方法进行预测，该方法会调用pipeline的每一个stage内的Transformer的transform方法来对输入的DataFrame/DataSet进行转换。
```
@Since("2.0.0")
  override def transform(dataset: Dataset[_]): DataFrame = {
    transformSchema(dataset.schema, logging = true)
    stages.foldLeft(dataset.toDF)((cur, transformer) => transformer.transform(cur))
  }
```
下面，我们先看看训练好的随机森林模型(RandomForestClassificationModel)在预测时做了些什么吧
```
override protected def transformImpl(dataset: Dataset[_]): DataFrame = {
    val bcastModel = dataset.sparkSession.sparkContext.broadcast(this)
    val predictUDF = udf { (features: Any) =>
      bcastModel.value.predict(features.asInstanceOf[Vector])
    }
    dataset.withColumn($(predictionCol), predictUDF(col($(featuresCol))))
  }
```
重点来了，终于找到前面说的broadcast的'罪魁祸'了，每次预测时，MLlib都会把模型广播到集群。这样做的好处是方便批处理，但对于小计算量，压根不需要集群的预测场景这样的做法就有点浪费资源了：
1. 每次预测都广播显然太多余。
2. 因为每次都广播，所以之前的广播变量也会逐渐回收，在回收时，又反过来影响预测的性能。
## 解决办法
从上述代码中可以看到，RandomForestClassificationModel 预测最根本的地方是在于调用predict方法，输入是一个Vector。看看predict干了什么
```
override protected def predict(features: FeaturesType): Double = {
    raw2prediction(predictRaw(features))
  }
```
predict分为两步走: 
```
override protected def predictRaw(features: Vector): Vector = {
    // TODO: When we add a generic Bagging class, handle transform there: SPARK-7128
    // Classifies using majority votes.
    // Ignore the tree weights since all are 1.0 for now.
    val votes = Array.fill[Double](numClasses)(0.0)
    _trees.view.foreach { tree =>
      val classCounts: Array[Double] = tree.rootNode.predictImpl(features).impurityStats.stats
      val total = classCounts.sum
      if (total != 0) {
        var i = 0
        while (i < numClasses) {
          votes(i) += classCounts(i) / total
          i += 1
        }
      }
    }
    Vectors.dense(votes)
  }

  override protected def raw2probabilityInPlace(rawPrediction: Vector): Vector = {
    rawPrediction match {
      case dv: DenseVector =>
        ProbabilisticClassificationModel.normalizeToProbabilitiesInPlace(dv)
        dv
      case sv: SparseVector =>
        throw new RuntimeException("Unexpected error in RandomForestClassificationModel:" +
          " raw2probabilityInPlace encountered SparseVector")
    }
  }

```
这两个方法的输入和输出均为vector，那么我们如果把这两个方法反射出来直接用在预测的特征向量上是不是就可以了？答案是肯定的。
注意其中的`raw2probability`在Spark2.3中的RandomForestClassificationModel中，签名变为了`raw2probabilityInPlace`
## 全面绕开pipeline
前面解决了分类器预测的性能问题，另外一个问题就来了。输入的特征向量怎么来呢？在一个MLlib Pipeline流程中，分类器预测只是最后一步，前面还有多种多样的特征加工节点。我尝试了将一个pipeline拆解成两个，一个用于特征加工，一个用于分类预测。用第一个pipeline加工特征，只绕开第二个，性能显然是提升了，但还没达到预期效果。于是，我有了另外一个想法：全面绕开pipeline，对pipeline的每一步，都避免调用原生transform接口。这样的弊端就是，必须重写pipeline的每一步预测方法，然后人肉还原pipeline的预测流程。流程大致跟上面类似。
例如：OneHot(说句题外话，这东西在Spark2.3之前的版本是有bug的，详情参考官方文档)。
OneHotEncoderModel的transform方法如下:
```
@Since("2.3.0")
  override def transform(dataset: Dataset[_]): DataFrame = {
    val transformedSchema = transformSchema(dataset.schema, logging = true)
    val keepInvalid = $(handleInvalid) == OneHotEncoderEstimator.KEEP_INVALID

    val encodedColumns = $(inputCols).indices.map { idx =>
      val inputColName = $(inputCols)(idx)
      val outputColName = $(outputCols)(idx)

      val outputAttrGroupFromSchema =
        AttributeGroup.fromStructField(transformedSchema(outputColName))

      val metadata = if (outputAttrGroupFromSchema.size < 0) {
        OneHotEncoderCommon.createAttrGroupForAttrNames(outputColName,
          categorySizes(idx), $(dropLast), keepInvalid).toMetadata()
      } else {
        outputAttrGroupFromSchema.toMetadata()
      }

      encoder(col(inputColName).cast(DoubleType), lit(idx))
        .as(outputColName, metadata)
    }
    dataset.withColumns($(outputCols), encodedColumns)
  }
```
里面对feature进行转换的关键代码行是 encoder...
```
private def encoder: UserDefinedFunction = {
    val keepInvalid = getHandleInvalid == OneHotEncoderEstimator.KEEP_INVALID
    val configedSizes = getConfigedCategorySizes
    val localCategorySizes = categorySizes

    // The udf performed on input data. The first parameter is the input value. The second
    // parameter is the index in inputCols of the column being encoded.
    udf { (label: Double, colIdx: Int) =>
      val origCategorySize = localCategorySizes(colIdx)
      // idx: index in vector of the single 1-valued element
      val idx = if (label >= 0 && label < origCategorySize) {
        label
      } else {
        if (keepInvalid) {
          origCategorySize
        } else {
          if (label < 0) {
            throw new SparkException(s"Negative value: $label. Input can't be negative. " +
              s"To handle invalid values, set Param handleInvalid to " +
              s"${OneHotEncoderEstimator.KEEP_INVALID}")
          } else {
            throw new SparkException(s"Unseen value: $label. To handle unseen values, " +
              s"set Param handleInvalid to ${OneHotEncoderEstimator.KEEP_INVALID}.")
          }
        }
      }

      val size = configedSizes(colIdx)
      if (idx < size) {
        Vectors.sparse(size, Array(idx.toInt), Array(1.0))
      } else {
        Vectors.sparse(size, Array.empty[Int], Array.empty[Double])
      }
    }
  }
```
encoder里面关键的是这个udf，将其抠出重写之后直接作用于特征向量。

## 效果
经过测试，全面绕开pipeline之后，响应时间下降到16ms左右。(2000条query平均响应时间)，且不再有抖动。
