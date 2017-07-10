---
layout: post 
author: shalou
title:  "prometheus metric type"   
category: 容器技术
tag: ["prometheus"]
---


Prometheus客户库提供了四个核心的metrics类型，分别为Counter、Gauge、Histogram和Summary

## 1. Counter(计数器)
counter 是一个累计度量指标，它是一个只能递增的数值。计数器主要用于统计服务的请求数、任务完成数和错误出现的次数等等

```golang
type Counter interface {
    Metric
    Collector

    // Inc increments the counter by 1. Use Add to increment it by arbitrary
    // non-negative values.
    Inc()
    // Add adds the given value to the counter. It panics if the value is <
    // 0.
    Add(float64)
}
```
当需要增加次数时，调用Inc()函数即可

<!-- more -->

## 2. Gauge(尺度)

gauge是一个度量指标，它表示一个既可以递增, 又可以递减的值。

计量器主要用在类似于温度、当前内存使用量等，也可以统计当前服务运行随时增加或者减少的Goroutines数量

```
type Gauge interface {
    Metric
    Collector

    // Set sets the Gauge to an arbitrary value.
    Set(float64)
    // Inc increments the Gauge by 1. Use Add to increment it by arbitrary
    // values.
    Inc()
    // Dec decrements the Gauge by 1. Use Sub to decrement it by arbitrary
    // values.
    Dec()
    // Add adds the given value to the Gauge. (The value can be negative,
    // resulting in a decrease of the Gauge.)
    Add(float64)
    // Sub subtracts the given value from the Gauge. (The value can be
    // negative, resulting in an increase of the Gauge.)
    Sub(float64)

    // SetToCurrentTime sets the Gauge to the current Unix time in seconds.
    SetToCurrentTime()
}
```
调用Set(float64)可以设置其值

## 3. Histogram(直方图)
Histogram对观察结果(通常是请求持续时间或者响应大小)进行采样，并在可配置的桶中对其进行统计。它还提供所有观察值的总和

在一次数据采样过程中，直方图会提供以下数据（这些数据以\<basename\>开头）

* 观察桶累计计数器，命名为\<basename\>_bucket={le="\<upper inclusive bound\>"}
* 观察值的总和，命名为\<basename
>_sum
* 观察的总次数\<basename\>_count（等于\<basename\>_bucket{le="+Inf"}）


对Histogram的理解：Prometheus会针对度量指标进行histogram统计，生成三个度量指标数据，即_bucket, _sum, 和_count。对于<basename>_bucket: 会有三个数据输入，一个是基准值，一个是每次增长的步长，一个是横坐标的长度。

举一个具体的例子，统计Http请求的处理延迟时间，延迟的基准值为300ms，步长为100ms，横坐标的值为5，则会生成5个桶。这5个桶分别代表延迟低于300ms、400ms、500ms、600ms、700ms的请求次数。此外还会生成http延迟的总和

```
histogram: <
  sample_count: 2000
  sample_sum: 5996820.60000000001
  bucket: <
    cumulative_count: 383
    upper_bound: 300
  >
  bucket: <
    cumulative_count: 729
    upper_bound: 400
  >
  bucket: <
    cumulative_count: 1002
    upper_bound: 500
  >
  bucket: <
    cumulative_count: 1278
    upper_bound: 600
  >
  bucket: <
    cumulative_count: 1632
    upper_bound: 700
  >
>
```
此外还有一个桶应该为\<basename\>_bucket{le="+Inf"}，即Http延迟数低于无穷大的请求次数，等于\<basename\>_count

histogram_quantile这个函数经常用于计算histogram的百分比数目，比如我们想知道90%的http请求低于某个延迟数，可以使用histogram_quantile(0.9, histogram)

## 4. Summary(总结)

类似histogram，summary观察样本值(使用场景类似：请求持续时间和响应大小)。它也提供观察的总计数和所有观察值的总和。同时它可以在滑动时间窗口上计算可配置的分位数。

在一次数据采样过程中，Summary会提供以下数据（这些数据以\<basename\>开头）

* 观察时间的φ-quantiles (0 ≤ φ ≤ 1), 显示为[basename]{分位数="[φ]"}
* [basename]_sum， 是指所有观察值的总和
* [basename]_count, 是指已观察到的事件计数值

summary默认的分位数为0.5、0.9、0.99，用户可以手动指定分位数的值

```
summary: <
 sample_count: 1000
 sample_sum: 29969.50000000001
 quantile: <
   quantile: 0.5
   value: 31.1
 >
 quantile: <
   quantile: 0.9
   value: 41.3
 >
 quantile: <
   quantile: 0.99
   value: 41.9
 >
>
```

即表示一共采集了1000个样本，这些样本总和为29969.50000000001，其中50%的样本值低于31.1，90%的样本值低于41.3，99%的样本值低于41.9
