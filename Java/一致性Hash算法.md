## 一致性hash算法的介绍

一致性hash算法是分布式应用中的常用且好用的算法、常用于负载均衡和分库分表上[redis等缓存]，
通常对于正是意义上的分布式应用来说，扩容和收缩是一个半自动化的过程，在此期间，应用基本上是可用的，所以不能发生大规模动荡的意外，
为了最小化潜在的影响，一致性hash算法扮演了极为重要的角色。


### 1.基本场景

比如有N台缓存服务器，那么如何将对象映射到N台服务器上呢，你可能会采用Hash(key) % N, 然后均匀的分布到N个服务器上

缺点:

- 如果一个服务器down了，所有映射到此服务器端的缓存全部是失效，这个时候cache是N - 1台

- 如果增加了一台服务器，此时的缓存服务器基本上大部分失效，这个时候cache是N + 1台

这意味着突然间几乎所有的缓存都将失效，对于高并发的情况下，就是一场灾难，因此使用一致性hash算法来解决这个问题

### 2.算法原理

一致性hash是一种算法，简单的来说，增加或者移除一个cache时，他能够尽可能小的改变已存在key的映射关系，尽可能的满足单调性的要求

- 环形hash空间

考虑到通用的hash算法，都是将value映射到32位的key值，也就是0-2^32-1的数值空间，我们将这个空间想象成一个首0-尾2^32-1相接的圆环

- 把对象映射到hash空间里

把对象通过hash算法，映射出一个hash值

- 把cache映射到hash空间

即将服务器进行hash到数值空间中，一般使用机器的ip或者机器名作为hash输入

- 将对象映射到cache中

cache和hash都映射到hash空间，接下来就是在这个空间中，将对象的key值出发，顺时针方法出发，遇到的第一个cahe就是这个对象存储的cache

- 考虑cache的变动，通过hash最不能解决的就是不能满足单调性，当cache发生变市，cache会失效，进而对后台服务器造成巨大冲击

    - 移除cache，如果cache移除的时候，只需要继续沿着顺时针的方向继续遍历到下一个cache

    - 增加cache，再考虑添加cache的情况，会被重新映射，原来分布的不会受到影响


### 3.虚拟节点

考虑到hash算法的另一个指标平衡性，平衡性是指hash的结果尽可能分布到所有的缓存中去，这样可以使得所有的缓存空间都得到利用

hash算法是不能保证绝对的平衡，如果cache少的时候，对象是不能均匀分布到cache上的，因此为了解决这个问题，引入了虚拟节点的概念

虚拟节点是实际节点在hash空间上的一个替代值，一个实际节点对应若干个虚拟节点，这个对应个数也称为复制个数，虚拟节点在hash空间中以hash值排列

虚拟节点的生成可以使用节点ip地址加数字后缀的方式，假设第一个节点ip为192.168.3.8，则计算第一个节点的hash值为hash(192.168.3.8)
如果引入虚拟节点，则计算方式为如下

hash(192.168.3.8#1) --> node1

hash(192.168.3.8#2) --> node2


### 4.代码实现

```java
package com.evision.common.loadbalance;

/**
 * 负载均衡器
 *
 * @author asheng
 * @since 2020/9/10
 */
public interface LoadBalance {

    /**
     * 根据key选择服务器
     *
     * @param key 服务器选择key
     * @return 服务器地址
     */
    String choose(String key);

}
```

```java
package com.evision.common.loadbalance;

import com.evision.common.Assert;

import java.util.LinkedList;
import java.util.List;
import java.util.SortedMap;
import java.util.TreeMap;

/**
 * 一致性hash算法[带有虚拟节点]
 *
 * @author asheng
 * @since 2020/9/1
 */
public class ConsistentHashLoadBalance implements LoadBalance {

    /** 已经归类的节点map */
    protected final SortedMap<Integer, String> nodeMap = new TreeMap<>();

    /** 实际节点 */
    private final List<String> realNodes = new LinkedList<>();

    /** 虚拟节点数 */
    private final int virtualNodeCount;

    public ConsistentHashLoadBalance(List<String> nodes, int virtualNodeCount) {
        Assert.assertNotEmpty(nodes, "server nodes can not be empty");
        this.virtualNodeCount = virtualNodeCount;
        for (String node : nodes) {
            realNodes.add(node);
            for (int i = 0; i < virtualNodeCount; i ++) {
                String virtualNode = node + "##" + i;
                int hash = getHash(virtualNode);
                nodeMap.put(hash, virtualNode);
            }
        }
    }

    @Override
    public String choose(String key) {
        int hash = getHash(key);
        // 获取比此key大的所有节点
        SortedMap<Integer, String> subMap = nodeMap.tailMap(hash);

        String serverVirtualNode;
        if (subMap.isEmpty()) {
            // 如果节点为空，则表示此环回归到了终点，选取第一个
            Integer idx = nodeMap.firstKey();
            serverVirtualNode = nodeMap.get(idx);
        } else {
            // 获取里此key最近的一个节点
            Integer idx = subMap.firstKey();
            serverVirtualNode = subMap.get(idx);
        }
        return serverVirtualNode.substring(0, serverVirtualNode.indexOf("##"));
    }

    /**
     * 使用FNV1_32 Hash算法
     *
     * @param key 需要做hash的key
     * @return hash结果
     */
    protected int getHash(String key) {
        final int p = 16777619;
        int hash = (int) 2166136261L;
        for (int i = 0; i < key.length(); i++) {
            hash = (hash ^ key.charAt(i)) * p;
        }
        hash += hash << 13;
        hash ^= hash >> 7;
        hash += hash << 3;
        hash ^= hash >> 17;
        hash += hash << 5;
        return hash < 0 ? Math.abs(hash) : hash;
    }

    public int getVirtualNodeCount() {
        return virtualNodeCount;
    }

    public List<String> getRealNodes() {
        return realNodes;
    }
}
```