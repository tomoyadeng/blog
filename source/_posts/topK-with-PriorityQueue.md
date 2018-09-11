---
title: 使用 PriorityQueue 求解 Top K 问题
date: 2018-05-13 14:27:59
tags: Java
---

### 0x00 问题描述

这两天在看一些面试题的时候，遇到一个问题:

> 有N(N>>10000)个整数,求出其中的前K个最大的数

在网上搜了下，大概有三种解决思路：

+ 排序：这种方式最好理解，但是时间复杂度较高(使用快排,O(NlogN))
+ 堆： 维护一个有边界的小顶堆(O(NlogK))
+ 位图： 理解较为困难 (O(N))

自己动手试了试第二种思路在Java中的实现(泛型版本)

### 0x01 Java实现

在 Java 中，PriorityQueue 类实现了堆这种数据结构，可以用来求解Top K 问题。

整个算法的思想就是： 通过PriorityQueue实现一个有界的堆，逐个向堆中添加元素，当元素个数超过边界时，淘汰堆顶元素

<!-- more -->

```java
package com.tomoyadeng.javabeginner.interview;

import java.util.*;
import java.util.stream.Stream;

public class TopK<T> {

    /**
     * 堆的边界，Top K 问题中的 K
     */
    private final int boundary;

    /**
     * 优先队列，用来构造一个有界的堆
     */
    private final PriorityQueue<T> boundaryHeap;


    /**
     * 通过自定义边界 boundary 可以求解 top K 问题
     * 通过自定义比较器 comparator 可以控制求解 top K 大 还是 top K 小
     * @param boundary 边界 K
     * @param comparator 数据比较器
     */
    public TopK(int boundary, Comparator<T> comparator) {
        this.boundary = boundary;
        boundaryHeap = new PriorityQueue<>(boundary, comparator);
    }

    /**
     * 求解数据流中的top K， 将结果写入List中
     * @param dataStream 数据流
     * @param results top K 结果
     */
    public void topK(Stream<T> dataStream, List<T> results) {
        dataStream.forEach(this::add);

        while (!boundaryHeap.isEmpty()) {
            results.add(boundaryHeap.poll());
        }
    }

    /**
     * 向有界堆中添加元素的帮助方法
     * @param t 待添加数据
     */
    private void add(T t) {
        boundaryHeap.add(t);
        if (boundaryHeap.size() > boundary) {
            boundaryHeap.poll();
        }
    }
}

```

### 0x02 测试

直接写一个main函数进行测试

```java
public static void main(String[] args) {
    // 构造一个 范围为 [0, 2^30] 的 Integer 流，通过limit可以控制大小
    final int upLimit = 1 << 30;
    Stream<Integer> stream = Stream.generate(Math::random)
            .map(d -> d * upLimit)
            .map(d -> (int) Math.round(d))
            .limit(100_000_000);

    // 将 (o1, o2) -> (o1 - o2) 换成 (o1, o2) -> (o2 - o1) 可以求解 top K 小
    TopK<Integer> topK = new TopK<>(10, (o1, o2) -> (o1 - o2));

    List<Integer> results = new ArrayList<>();

    long startTime = System.currentTimeMillis();

    topK.topK(stream, results);

    long endTime = System.currentTimeMillis();

    System.out.println("results: " + results);
    System.out.println("cost: " + (endTime - startTime) / 1000.0);
}
```

1亿数据测试结果：

> results: [1073741717, 1073741721, 1073741740, 1073741747, 1073741768, 1073741781, 1073741785, 1073741791, 1073741792, 1073741813]
> cost: 7.656

### 0x03 优点分析

在输入数据流是一个惰性流(不需要一次性将全部数据加载到内存)的情况下，这种方式速度较快且占用最少的内存，内存中只需要维护一个固定大小的堆即可
