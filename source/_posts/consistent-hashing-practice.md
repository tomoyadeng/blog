---
title: 一致性哈希算法的一次实践
date: 2018-06-09 22:18:17
tags: Java
categories: Java
---

最近在分析一个需求，需要开发一个采集器的调度框架，实现采集器的注册，离线以及采集任务分配(负载均衡)。

采集器用于登录到网络设备上采集数据，部分运营商考虑到设备性能问题，会限制同时只能有一个用户登录设备查询数据。那么在此限制下，分配采集任务时需要保证：

+ 对于同一设备的任务始终都落在同一个采集器上去执行，才能保证同一时刻对于同一设备不会有多个采集器采集。
+ 而且，需要保证在某个采集器失效离线时，之前落在该采集器上的设备列表需要均匀分布到剩下的采集器上去，不至于造成某一个采集器负载过大。

到这里，实现方案已经呼之欲出，这不就是解决分布式缓存问题的套路么 --- 一致性哈希算法，可以参考这篇文章进行了解 [《一致性哈希算法及其在分布式系统中的应用》](http://blog.codinglabs.org/articles/consistent-hashing.html)

<!-- more -->

### 0x01 接口定义

首先，对几个关键角色进行接口抽象

#### 网络设备

```java
/**
 * 网络设备
 */
public interface Device {
    /**
     * 获取设备IP
     * @return IP
     */
    String getIp();
}
```

#### 采集器

```java
public interface Collector {
    /**
     * 获取采集器IP
     * @return IP
     */
    String getIp();

    /**
     * 设置采集器IP
     * @param ip IP
     */
    void setIp(String ip);

    /**
     * 执行采集任务
     * @param device 采集对象--设备
     * @param commands 采集命令
     * @return 采集结果
     */
    Map<String, Object> collect(Device device, List<String> commands);
}
```

#### 集群

这里把采集器的调度框架抽象成集群，并且使用泛型来定义集群接口

```java
/**
 * 调度框架(可看作集群管理)
 */
public interface Cluster<T> {
    /**
     * 注册
     * @param t
     */
    void register(T t);

    /**
     * 离线
     * @param t
     */
    void offline(T t);

    /**
     * 负载均衡
     * @param ip 源IP
     * @return T
     */
    T choose(String ip);
}
```

### 0x02 算法实现

#### Hash算法选择

在选择设备对应的采集器时，需要对设备的IP进行hash计算。由于设备的IP前缀基本一致，使用默认的字符串hash算法会导致计算出来的hash值不够离散，只能落在hash环上很小的一段区间。因此需要重新选择一种hash算法，保证字符串hash的离散性。这里使用[FNV1_32_HASH](https://dustin.sallings.org/java-memcached-client/apidocs/net/spy/memcached/HashAlgorithm.html#FNV1_32_HASH)算法

```java
/**
    * FNV1_32_HASH
    *
    * @param str str
    * @return hash
    */
private static int rehash(String str) {
    final int p = 16777619;
    int hash = (int) 2166136261L;
    for (int i = 0; i < str.length(); i++) {
        hash = (hash ^ str.charAt(i)) * p;
    }
    hash += hash << 13;
    hash ^= hash >> 7;
    hash += hash << 3;
    hash ^= hash >> 17;
    hash += hash << 5;

    if (hash < 0) {
        hash = Math.abs(hash);
    }
    return hash;
}
```

#### 具体实现

```java

/** @author tomoya */
public class CollectorCluster implements Cluster<Collector> {

    /** 每个采集器定义虚拟节点个数 */
    private static final int VIRTUAL_NODE_NUMBER = 320;

    /** 采集器--所有虚拟节点hash值数组 映射关系 */
    @GuardedBy("clusterLock")
    private Map<Collector, int[]> registeredServers = new HashMap<>();

    /** hash环上的hash值--采集器 映射关系 */
    @GuardedBy("clusterLock")
    private TreeMap<Integer, Collector> hashRingMap = new TreeMap<>();

    private ReadWriteLock clusterLock = new ReentrantReadWriteLock();

    @Override
    public void register(Collector collector) {
        System.out.println("add server " + collector.toString());

        final Lock lock = clusterLock.writeLock();
        lock.lock();
        try {
            // 计算采集器所有虚拟节点的hash值，并将所有hash值注册到hash环上
            int[] nodesHash = new int[VIRTUAL_NODE_NUMBER];
            for (int i = 0; i < VIRTUAL_NODE_NUMBER; i++) {
                int hash = CollectorCluster.rehash(collector.getIp() + ":" + i);
                nodesHash[i] = hash;
                hashRingMap.put(hash, collector);
            }
            // 保存采集所有虚拟节点的hash值
            registeredServers.put(collector, nodesHash);
        } finally {
            lock.unlock();
        }
    }

    @Override
    public void offline(Collector collector) {
        System.out.println("delete server " + collector.toString());

        final Lock lock = clusterLock.writeLock();
        lock.lock();
        try {
            // 将该采集器所有虚拟节点的hash值从hash环上删除
            for (int hash : registeredServers.get(collector)) {
                hashRingMap.remove(hash);
            }
            // 删除采集器
            registeredServers.remove(collector);
        } finally {
            lock.unlock();
        }
    }

    @Override
    public Collector choose(String deviceIp) {
        final int hash = rehash(deviceIp);

        final Lock lock = clusterLock.readLock();
        lock.lock();
        try {
            // 逆时针找映射节点
            Map.Entry<Integer, Collector> entry = hashRingMap.floorEntry(hash);
            Collector collector =
                    entry == null ? hashRingMap.lastEntry().getValue() : entry.getValue();

            System.out.println(deviceIp + " --> " + collector);
            return collector;
        } finally {
            lock.unlock();
        }
    }
}
```

### 0x03 测试

首先实现一个具体的采集器类

```java
/**
 * @author tomoya
 */
public class DefaultCollector implements Collector {
    private String ip;

    DefaultCollector(String ip) {
        this.ip = ip;
    }

    @Override
    public String getIp() {
        return ip;
    }

    @Override
    public void setIp(String ip) {
        this.ip = ip;
    }

    @Override
    public Map<String, Object> collect(Device ne, List<String> commands) {
        return new HashMap<>(commands.size());
    }

    @Override
    public String toString() {
        return "DefaultCollector{" +
                "ip='" + ip + '\'' +
                '}';
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        DefaultCollector that = (DefaultCollector) o;
        return Objects.equals(ip, that.ip);
    }

    @Override
    public int hashCode() {

        return CollectorCluster.rehash(ip);
    }
}
```

然后直接在CollectorCluster类中增加一个main函数来测试

```java
public static void main(String[] args) {
    CollectorCluster cluster = new CollectorCluster();

    // 注册5个采集器
    List.of("192.168.0.1", "192.168.0.2", "192.168.0.3", "192.168.0.4", "192.168.0.5")
            .stream()
            .map(DefaultCollector::new)
            .forEach(cluster::register);

    String ipPrefix = "136.10.1.";
    // 20个设备 进行负载均衡
    Stream.iterate(1, i -> i + 1).limit(20).map(i -> ipPrefix + i).forEach(cluster::choose);

    System.out.println("============");
    // 离线一个采集器
    cluster.offline(new DefaultCollector("192.168.0.5"));
    // 20个设备 再次进行负载均衡
    Stream.iterate(1, i -> i + 1).limit(20).map(i -> ipPrefix + i).forEach(cluster::choose);
}
```
