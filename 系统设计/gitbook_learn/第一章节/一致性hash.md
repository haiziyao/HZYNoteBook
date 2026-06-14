---
tags:
  - 系统设计
description: 一致性hash
color: cyan
---
什么是一致性hash算法,为什么需要一致性hash算法?

#### 引入
对于分布式系统,我们要求要将id均匀分布到n台服务器,那么就会有对n进行hash的操作,但是如果某个节点宕机了,那么我们就需要进行***再hash***的操作.

对于再hash,几乎大部分的映射关系都要改变,而且我们完全不能保证新hash是对n-1的平均分布
如果有部分缓存使用的不是分布式缓存,那么就会造成大面积缓存失效的问题.

所以,我们需要一个再hash稳定的一致性hash算法

#### hash空间与hash环

我们把所有的 数据 构成一个环状空间,想象出一个环形跑道,我只需要在跑到上放 n 个标识点,对于每一个数据,都按照 顺时针 寻找 标识, 最先找到的标识点就是 分配到的服务器.

这样,我们就只需要设计 '如何分配这些标识点的位置' 就可以了.

##### 节点宕机?
如果节点宕机,那么按照顺时针会寻找下一个节点,仅仅一小段的 数据映射受到影响,其他很多都没有任何影响.

##### 均衡?
如果新加一个节点/宕机,那么一定会造成不平衡,解决办法是:***引入虚拟节点***

新加一个节点: 我们把节点A分配为 A_0,A_1,A_2等等,按照一定分配方式分布在hash环上.
节点宕机: 同上,B_0.B_1等会同时挂机

可以很清楚的想明白,引入虚拟节点之后可以很好解决均衡的问题.

##### 总结

一致性 Hash 是一种让分布式系统在节点增减时尽量少迁移数据的 hash 分片算法。它通过 hash 环和虚拟节点，在稳定性和负载均衡之间取得平衡。


``` java
public class ConsistentHashing {  
  
    private Set<String> physicalNodes = new TreeSet<String>() {  
        {  
            add("192.168.1.101");  
            add("192.168.1.102");  
            add("192.168.1.103");  
            add("192.168.1.104");  
            add("192.168.1.135");  
            add("192.168.1.157");  
        }  
    };  
  
    //虚拟节点  
    private final int VIRTUAL_COPIES = 1048576; // 物理节点至虚拟节点的复制倍数  
    private TreeMap<Long, String> virtualNodes = new TreeMap<>(); // 哈希值 => 物理节点  
  
    // 32位的 Fowler-Noll-Vo 哈希算法  
    private static Long FNVHash(String key) {  
        final int p = 16777619;  
        Long hash = 2166136261L;  
        for (int idx = 0, num = key.length(); idx < num; ++idx) {  
            hash = (hash ^ key.charAt(idx)) * p;  
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
  
    // 根据物理节点，构建虚拟节点映射表  
    public ConsistentHashing() {  
        System.out.println("初始化开始");  
        for (String nodeIp : physicalNodes) {  
            addPhysicalNode(nodeIp);  
        }  
        System.out.println("初始化结束");  
    }  
  
    // 添加物理节点  
    public void addPhysicalNode(String nodeIp) {  
        for (int idx = 0; idx < VIRTUAL_COPIES; ++idx) {  
            String virtualIp = nodeIp + "#" + idx;  
            long hash = FNVHash(virtualIp);  
            virtualNodes.put(hash, nodeIp);  
        }  
    }  
  
    // 删除物理节点  
    public void removePhysicalNode(String nodeIp) {  
        for (int idx = 0; idx < VIRTUAL_COPIES; ++idx) {  
            long hash = FNVHash(nodeIp + "#" + idx);  
            virtualNodes.remove(hash);  
        }  
    }  
  
    // 查找对象映射的节点  
    public String getObjectNode(String object) {  
        long hash = FNVHash(object);  
  
        // 找到第一个 >= hash 的虚拟节点  
        Map.Entry<Long, String> entry = virtualNodes.ceilingEntry(hash);  
  
        // 如果 hash 比环上所有节点都大，就回到环头  
        if (entry == null) {  
            entry = virtualNodes.firstEntry();  
        }  
  
        return entry.getValue();  
    }  
  
    // 统计对象与节点的映射关系  
    public void dumpObjectNodeMap(String label, int objectMin, int objectMax) {  
        if (objectMin > objectMax) {  
            throw new IllegalArgumentException("objectMin 不能大于 objectMax");  
        }  
  
        Map<String, Integer> objectNodeMap = new TreeMap<>();  
  
        for (int object = objectMin; object <= objectMax; object++) {  
            String nodeIp = getObjectNode(String.valueOf(object));  
  
            // 等价于：如果不存在就设置为 1，如果存在就 +1            objectNodeMap.merge(nodeIp, 1, Integer::sum);  
        }  
  
        double totalCount = (double) objectMax - objectMin + 1;  
  
        System.out.println("======== " + label + " ========");  
  
        for (Map.Entry<String, Integer> entry : objectNodeMap.entrySet()) {  
            double percent = 100.0 * entry.getValue() / totalCount;  
  
            System.out.printf(  
                    "IP=%s: COUNT=%d, RATE=%.2f%%%n",  
                    entry.getKey(),  
                    entry.getValue(),  
                    percent  
            );  
        }  
    }  
  
    public static void main(String[] args) {  
        ConsistentHashing ch = new ConsistentHashing();  
  
        // 初始情况  
        ch.dumpObjectNodeMap("初始情况", 0, 65536);  
  
        // 删除物理节点  
        ch.removePhysicalNode("192.168.1.103");  
        ch.dumpObjectNodeMap("删除物理节点", 0, 65536);  
  
        // 添加物理节点  
        ch.addPhysicalNode("192.168.1.108");  
        ch.dumpObjectNodeMap("添加物理节点", 0, 65536);  
    }  
}
```

##### 执行结果

>初始化开始
初始化结束
======== 初始情况 ========
IP=192.168.1.101: COUNT=10821, RATE=16.51%
IP=192.168.1.102: COUNT=10815, RATE=16.50%
IP=192.168.1.103: COUNT=10983, RATE=16.76%
IP=192.168.1.104: COUNT=10930, RATE=16.68%
IP=192.168.1.135: COUNT=11069, RATE=16.89%
IP=192.168.1.157: COUNT=10919, RATE=16.66%
======== 删除物理节点 ========
IP=192.168.1.101: COUNT=13077, RATE=19.95%
IP=192.168.1.102: COUNT=12993, RATE=19.83%
IP=192.168.1.104: COUNT=13075, RATE=19.95%
IP=192.168.1.135: COUNT=13305, RATE=20.30%
IP=192.168.1.157: COUNT=13087, RATE=19.97%
======== 添加物理节点 ========
IP=192.168.1.101: COUNT=10893, RATE=16.62%
IP=192.168.1.102: COUNT=10808, RATE=16.49%
IP=192.168.1.104: COUNT=10861, RATE=16.57%
IP=192.168.1.108: COUNT=10899, RATE=16.63%
IP=192.168.1.135: COUNT=11169, RATE=17.04%
IP=192.168.1.157: COUNT=10907, RATE=16.64%

---

## 相关笔记

- [[限流算法]] — 分布式系统的另一核心主题：流量控制策略
- [[系统设计gitbook简介]] — 系统设计学习入口
