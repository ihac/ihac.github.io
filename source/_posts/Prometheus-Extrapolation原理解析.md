---
title: Prometheus Extrapolation原理解析
date: 2018-12-11 19:32:03
tags: [Prometheus]
---

在使用Prometheus的过程中，我们可能遇到过这样的问题：PromQL的delta、increase函数计算一段时间内metric的增长量，返回的结果可能是小数，即便该metric所有的采样值都是整数。为了验证这一点，可以拿`promhttp_metric_handler_requests_total{code="200"}`做测试：使用increase对其求一段时间内的增长量，多次尝试，并调整时间窗口大小，很容易就能看到返回小数。

如果我们查阅官方文档，可以发现delta、increase、rate函数的介绍中都有这么一段类似的话：*The delta is extrapolated to cover the full time range as specified in the range vector selector, so that it is possible to get a non-integer result even if the sample values are all integers*。这段话翻译过来主要有两层含义：首先，“对整数型metric求差值返回小数”是feature，不是bug；其次，产生小数的原因在于being extrapolated。不过，光从extrapolate的字面意思很难理解Prometheus的具体计算过程，本文主要根据源码以及社区讨论内容来对Prometheus extrapolation机制的原理进行剖析，并介绍delta、increase、rate函数的实际计算过程。

## 设计初衷

Prometheus引入extrapolation机制的主要初衷在于解决[align问题][count_with_prometheus]（或数据缺失问题，二者本质一样）。

当用户执行`delta`、`increase`或`rate`等函数时，Prometheus需要对用户指定时间范围内的metric求增值或增长速率。从用户角度来看，metric应当是一条在时间维度上持续延展的曲线（或折线），因此对一段时间范围的增值（或速率）的求解是非常直观的：直接取区间两个端点对应的数值，相减（并除以区间长度）即可得到增值（速率）。然而，从设计角度上来看，Prometheus显然不可能将metric用一条曲线（折线）来表示，其一是没有足够大的空间来存储线上无数个点，其二是没有足够多的计算资源能够支撑对多个metric的实时收集上报。

事实上，Prometheus将metric表示为在时间维度上等距（大多数情况下）的多个离散点，其距离即为我们熟知的`scrape interval`。基于此，我们大致可以猜到为什么会产生align问题：用户指定的区间基本上很难和metric区间对齐，即用户区间端点并没有对应的metric数值，因此我们预想的通过端点取值相减是无法实现的。

## 具体实现

下面简单介绍三种解决align问题的思路，它们有各自的优缺点，但在Prometheus社区看来，这三种思路应该是按从劣到优的顺序排列的（不置可否!!!）。

### native method

首先，非常直观地，在区间端点无法取到metric的情况下，我们可以向前取最近的点作为当前的metric值。

举个例子，假设我们采取到的metric点有：{t=1, v[t]=10}、{t=6, v[t]=12}、{t=11, v[t]=13}，求解[2, 12]区间内的变化速率。根据上述思路，不难求得rate = (v[12] - v[2]) / (12 - 2) ≈ (v[11] - v[1]) / (12 - 2) = 0.3。

看上去这种思路很符合我们的认知，但试想，如果我们要求[0,10]区间内的变化速率，得到的结果将是1.2（因为零点并没有采值，所以默认为0），这可能和实际速率（比如0.3）相差甚远。因此，在计算`rate`、`increase`或`delta`时，Prometheus并没有采取这种做法，而是通过估计的方法来求解。

### extrapolation (before v0.16.1)

Prometheus extrapolation思路其实很简单，将metric区间进行等比放大，直至匹配到用户指定的时间范围。

基于上述案例，假设我们需要求[0, 15]区间范围内的增值，此时求解流程变为了：increase = v[15] - v[0] ≈ (v[11] - v[1]) * (15 - 0) / (11 - 1) = 4.5，其变化速率rate = increase / (15 - 0) = 0.3。

不难看出，extrapolation本质上就是期望通过类似“线性拟合”的方式来对不存在的区间进行估计求值。因此，这种方法对`与时间呈线性关系`的metric具有非常良好的效果，对非线性metric的估计则效果不佳。不过Prometheus引入extrapolation本身就不是为了提供一个精确解，而是在无法求得真实解的情况下，提供一个尽可能可靠的参考值而已。

尽管如此，初版的extrapolation在实际使用上还是存在一定问题，具体参见[issue 581][issue_581]及下图。图一是用户对多个metric序列求和的结果，图二是对多个metric序列分别求速率，然后求和的结果。注意到图二出现了一个非常“刺眼”的尖峰，对应速率高达`80 / 4h`，显然这与实际情况不符（从06:00到11:00，总共才增长了不到20）。

![](https://cloud.githubusercontent.com/assets/538008/6586657/6f05aaa8-c77b-11e4-9fa0-5ebe79eca528.png)

造成这一现象的原因非常简单——有一个metric序列在10:40左右前是缺失的，此时对所有序列求和是没有问题的（缺失序列默认为0），但对这些序列求解速率时，extrapolation机制的引入使得缺失部分将基于线性延伸来进行填充，即Prometheus认为缺失部分应当保持未缺失部分的增长速率，显然这与实际情况相悖。因此，我们可以看到图二中出现的尖峰速率（`80 / 4h`）其实正是10:40后的增长速率，而非整个4h的实际增长速率。

### extrapolation (after v0.16.1)

为了解决上述问题，更好地应对数据缺失的情况，社区曾有过一段时间的讨论（[pr 1161][pr_1161], [pr 1245][pr_1245], [pr1295][pr_1295]），最终通过的方案是：限制extrapolation的作用域，即只对短时间内的缺失做估计，长时间的缺失将一律用0替代。具体逻辑可以参照下一章节的源码分析。

延续上述速率爆炸的问题，以10:50时刻为例，新版extrapolation在计算增值时，不会再将未缺失部分（时长为10m左右）放大成完整的4h，而只会在前面补齐短短几分钟（实际为未缺失部分的平均sample间隔 / 2），求得增值后再除以4h作为估计的速率。因此，速率不会再出现爆炸的现象。

## 源码解析

下面结合源码，介绍Prometheus v2.6.5计算`delta`、`increase`或`rate`的实际流程。如下所示，三种query function都通过一个函数`extrapolatedRate`实现，具体通过`isCounter`和 `isRate`两个参数来进行区分。

PS：读懂源码需要对Prometheus内部数据结构（Vector、Matrix）有一定的了解。对于不熟悉的读者，建议将Vector理解为单个metric在时间维度上的采样序列，而Matrix是多个同类Vector的聚合（用label进行区分）。

```go
func extrapolatedRate(vals []Value, args Expressions, enh *EvalNodeHelper, isCounter bool, isRate bool) Vector {
    ms := args[0].(*MatrixSelector)

    var (
        matrix     = vals[0].(Matrix)
        rangeStart = enh.ts - durationMilliseconds(ms.Range+ms.Offset)
        rangeEnd   = enh.ts - durationMilliseconds(ms.Offset)
    )
    // 遍历多个vector，分别求解delta/increase/rate。
    for _, samples := range matrix {
        // 忽略少于两个点的vector。
        if len(samples.Points) < 2 {
            continue
        }
        var (
            counterCorrection float64
            lastValue         float64
        )
        // 由于counter存在reset的可能性，因此可能会出现0, 10, 5, ...这样的序列，
        // Prometheus认为从0到5实际的增值为10 + 5 = 15，而非5。
        // 这里的代码逻辑相当于将10累计到了couterCorrection中，最后补偿到总增值中。
        for _, sample := range samples.Points {
            if isCounter && sample.V < lastValue {
                counterCorrection += lastValue
            }
            lastValue = sample.V
        }
        resultValue := lastValue - samples.Points[0].V + counterCorrection

        // 采样序列与用户请求的区间边界的距离。
        // durationToStart表示第一个采样点到区间头部的距离。
        // durationToEnd表示最后一个采样点到区间尾部的距离。
        durationToStart := float64(samples.Points[0].T-rangeStart) / 1000
        durationToEnd := float64(rangeEnd-samples.Points[len(samples.Points)-1].T) / 1000
        // 采样序列的总时长。
        sampledInterval := float64(samples.Points[len(samples.Points)-1].T-samples.Points[0].T) / 1000
        // 采样序列的平均采样间隔，一般等于scrape interval。
        averageDurationBetweenSamples := sampledInterval / float64(len(samples.Points)-1)

        if isCounter && resultValue > 0 && samples.Points[0].V >= 0 {
            // 由于counter不能为负数，这里对零点位置作一个线性估计，
            // 确保durationToStart不会超过durationToZero。
            durationToZero := sampledInterval * (samples.Points[0].V / resultValue)
            if durationToZero < durationToStart {
                durationToStart = durationToZero
            }
        }

        // *************** extrapolation核心部分 *****************
        // 将平均sample间隔乘以1.1作为extrapolation的判断间隔。
        extrapolationThreshold := averageDurationBetweenSamples * 1.1
        extrapolateToInterval := sampledInterval
        // 如果采样序列与用户请求的区间在头部的距离不超过阈值的话，直接补齐；
        // 如果超过阈值的话，只补齐一般的平均采样间隔。这里解决了上述的速率爆炸问题。
        if durationToStart < extrapolationThreshold {
            // 在scrape interval不发生变化、数据不缺失的情况下，
            // 基本都进入这个分支。
            extrapolateToInterval += durationToStart
        } else {
            // 基本不会出现，除非scrape interval突然变很大，或者数据缺失。
            extrapolateToInterval += averageDurationBetweenSamples / 2
        }
        // 同理，参上。
        if durationToEnd < extrapolationThreshold {
            extrapolateToInterval += durationToEnd
        } else {
            extrapolateToInterval += averageDurationBetweenSamples / 2
        }
        // 对增值进行等比放大。
        resultValue = resultValue * (extrapolateToInterval / sampledInterval)
        // 如果是求解rate，除以总的时长。
        if isRate {
            resultValue = resultValue / ms.Range.Seconds()
        }

        enh.out = append(enh.out, Sample{
            Point: Point{V: resultValue},
        })
    }
    return enh.out
}
```

[count_with_prometheus]: https://www.youtube.com/watch?v=67Ulrq6DxwA)  "Counting with Prometheus [I] - Brian Brazil, Robust Perception"
[issue_581]: https://github.com/prometheus/prometheus/issues/581  "Issue 581: `rate` and `delta` deal poorly with slow moving data."
[pr_1161]: https://github.com/prometheus/prometheus/pull/1161   "PR 1161: promql: Remove extrapolation from rate/increase/delta"
[pr_1245]: https://github.com/prometheus/prometheus/pull/1245   "PR 1245: promql: Limit extrapolation of delta/rate/increase"
[pr_1295]: https://github.com/prometheus/prometheus/pull/1295   "PR 1295: promql: Limit extrapolation of delta/rate/increase"

## 参考资料

[1] Counting with Prometheus [I] - Brian Brazil, Robust Perception ([url](https://www.youtube.com/watch?v=67Ulrq6DxwA))
[2] Issue 581: rate and delta deal poorly with slow moving data ([url](https://github.com/prometheus/prometheus/issues/581))
[3] PR 1161: promql: Remove extrapolation from rate/increase/delta ([url](https://github.com/prometheus/prometheus/pull/1161))
[4] PR 1245: promql: Limit extrapolation of delta/rate/increase ([url](https://github.com/prometheus/prometheus/pull/1245))
[5] PR 1295: promql: Limit extrapolation of delta/rate/increase ([url](https://github.com/prometheus/prometheus/pull/1295))
[6] Issue 3746: rate()/increase() extrapolation considered harmful ([url](https://github.com/prometheus/prometheus/issues/3746))
[7] Issue 3806: Proposal for improving rate/increase ([url](https://github.com/prometheus/prometheus/issues/3806))
