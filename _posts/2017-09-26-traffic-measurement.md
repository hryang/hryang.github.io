---
layout: post
title: Sample and Hold - 基于采样的指标测量算法
---

<head>
    <script type="text/javascript"
            src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
    </script>
</head>

## 问题定义

在一个多租户的环境中，总是会出现用户突发的请求超过系统能处理的能力。在此种情况下，系统需要进行流控，丢弃一些请求，避免自身被压垮。好的流控系统，需要做到以下几点：

* 公平。比如少数用户产生了大量的请求，对系统造成很大压力，那么这部分用户的请求应该被流控。而其他正常访问的用户不应当被流控。因此流控本质上是一个概率问题，为每一个被流控的实体计算不同的概率值，按不同的概率拒绝请求。

* 及时。特别是短时间内请求大量增加时，要能快速的识别并启动流控。流控的信息要被系统的各个模块获取分析。比如，前端服务器获取流控信息后，在请求刚进入系统时就开始流控，减小对系统的影响；同时，master也需要获取流控信息，作为集群级别负载均衡的评估因素。

我们把统计和收集相关信息并计算被流控概率的单位称之为流控实体。例如在一个分布式数据库服务里，每一个table甚至每一个partition key可以看做流控实体。或者在分布式对象存储服务中，每个blob都是流控实体。当table和blob的数量非常多时，我们无法追踪每一个流控实体的资源使用情况，如何快速准确的找到需要被流控的实体是一个很有挑战的问题。

## Sample and Hold

当要统计的实体数量非常大时，通常使用采样的方式来进行统计。在一个measurement interval中，周期性的采样，获取统计信息。这种方式虽然降低了信息统计的成本，但是需要较长的measurement interval，才能识别出资源消耗大的实体。无法处理短时间请求压力暴增的场景。

[Sample and Hold](https://dl.acm.org/citation.cfm?id=633056)这篇论文提出了一种新的traffic measurement方法。对于任何请求，计算概率来决定该请求是否被采样。如果请求被采样，则将该请求所属的流控实体信息放到一个track table里。以后属于该流控实体的每个请求的信息都被统计。

术语定义

* C：系统能承担的最大流量，in bytes
* T：当流量超过T后，则被定义为large flow，可能需要被流控，in bytes
* O：Oversampling factor
* p：任意一个字节被采样的概率
* s：flow 实际发送的流量，in bytes
* c：flow 被统计的流量，in bytes

> 注： 这里的流量是广义的概念，可以是网络流量，也可以是请求数。

算法基本思路推理：

* 一共有 \\(n = \frac{C}{T}\\) 个 large flow。如果我们要确保 \\(n\\) 个 large flow 都被统计到，至少需要采样 \\(n\\) 次。考虑到 oversampling factor，在一个测量区间内，一共需要 \\(O * n\\) 次采样。所以 \\(p = \frac{O*n}{C} = \frac{O}{T}\\)
* 大小为 s 的请求，被采样的概率为 \\(1 - (1 - p)^s\\)，近似为 \\(p * s\\)
* Large flow 没有被识别的概率（false negative）：\\((1-p)^T = (1-\frac{O}{T})^T = e^{-O}\\)
* flow 实际发送的流量和被统计的流量差：在flow被第一次采样之前，miss的字节数服从[几何概率分布](https://zh.wikipedia.org/wiki/幾何分佈)，miss x 字节的概率为 \\(p=(1-p)^x*p\\)。因此 \\(E[c-s]=\frac{1}{p}\\)，\\(SD[c-s] = \frac{\sqrt{1-p}}{p}\\)



