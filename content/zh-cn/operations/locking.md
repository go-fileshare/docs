---
title: "一个映像，多个协议，一把锁"
weight: 30
description: "为什么通过多个协议提供的映像要被包裹在一把共享锁中，以及这层包装保留了什么。"
tags: [运维, 锁]
---

这个系列中的每个服务器都会对驱动本身串行化，因为
[`go-filesystems/interface`](https://github.com/go-filesystems/interface)
对并发的基于路径的调用不作任何保证，而它们各自持有
**自己的**锁，对其他锁一无所知。

因此，同时通过三个协议提供同一个映像，就会在任何地方都没有
共同的锁。所以映像在这里被包裹**一次**，同一层包装
交给所有协议使用。

## 如实陈述测量结果，以免夸大 {#stated-as-measured-so-as-not-to-overstate-it}

在 `-race` 下，八个 goroutine 通过**未包装**的 fat32 驱动写入，
目前**不会产生竞争**。

这是对契约的保险，而不是对缺陷的复现：ext4 和 ntfs
不是 fat32，而且驱动更新不是本程序应当
重新审计的事情。

## 这层包装会透传各项能力 {#the-wrapper-carries-capabilities-through}

它不会隐藏这些能力：

- 响应 `Opener` 的驱动会得到一个**带锁的 `File`**；
- 其 `File` 响应 `WritableFile` 的驱动会保留其**定位写入**。

在这个系列的其他地方，这些能力与整文件回退方案之间的差异经测量约为
**~70×**，因此抹去这些能力的包装
会是伪装成安全的性能缺陷。

## 在 `--isolate` 下，这把锁无能为力 {#under-isolate-the-lock-cannot-help}

子进程自行打开映像，因此子进程之间没有共享的包装。
这就是通过两个协议以可写方式提供的映像会被
[拒绝]({{< relref "/operations/isolation.md#the-rule-that-makes-it-honest" >}})，而不是悄悄地
在没有锁的情况下提供服务的原因。
