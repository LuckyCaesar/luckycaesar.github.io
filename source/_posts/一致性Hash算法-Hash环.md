---
title: 一致性Hash算法-Hash环
tags: [算法，一致性Hash，Hash环]
index_img: /img/hash-ring.jpg
date: 2024-05-28 16:00:00
---

一致性哈希算法（Consistent Hashing Algorithm）是一种哈希算法，它在分布式系统中用于存储和检索数据。

可以看看这篇文章简单理解学习一下：[一致性哈希算法-散列及Hash环相关的一些理解](https://segmentfault.com/a/1190000040422632)

以下是一个简单的Java实现：

```java
public class ConsistencyHash {

    private static final char[] CHAR_ARR = new char[]{
            'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm',
            'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z',
            'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M',
            'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z',
            '1', '2', '3', '4', '5', '6', '7', '8', '9', '0'};

    public static void main(String[] args) {
        Ring ring = buildRing();
        ArrayList<Node> virtualNodeList = new ArrayList<>();
        ArrayList<Node> actualNodeList = new ArrayList<>();
        for (Node node : ring.table) {
            if (node != null) {
                if (node.isVirtual) {
                    virtualNodeList.add(node);
                } else {
                    actualNodeList.add(node);
                }
            }
        }
        System.out.println("virtualNodeList: ");
        for (Node node : virtualNodeList) {
            System.out.println(node.name + ", index=" + node.index);
        }
        System.out.println();
        System.out.println("actualNodeList: ");
        for (Node node : actualNodeList) {
            System.out.println(node.name + ", index=" + node.index + ", size=" + node.dataList.size());
        }
    }

    private static Ring buildRing() {
        // 数据倾斜：环的容量尽量不要比节点（包括虚拟节点）的数量大太多，或者尽可能的增加节点（或虚拟节点）
        // 此处节点+虚拟节点的数量刚好占满整个环，所以数据一定是很均匀的
        Ring ring = new Ring(8, true);
        ring.addNode("node1");
        ring.addNode("node2");
        ring.addNode("node3");
        ring.addNode("node4");
        Random random = new Random();
        for (int i = 0; i < 18_0000; i++) {
            char c1 = CHAR_ARR[random.nextInt(62)];
            char c2 = CHAR_ARR[random.nextInt(62)];
            char c3 = CHAR_ARR[random.nextInt(62)];
            char c4 = CHAR_ARR[random.nextInt(62)];
            char c5 = CHAR_ARR[random.nextInt(62)];
            char c6 = CHAR_ARR[random.nextInt(62)];
            ring.addData(new String(new char[]{c1, c2, c3, c4, c5, c6}));
        }
        return ring;
    }

    static class Ring {
        Node[] table;
        int capacity;
        boolean needVirtual;// 是否需要自动映射虚拟节点

        public Ring(int capacity, boolean needVirtual) {
            this.capacity = capacity;
            this.needVirtual = needVirtual;
        }

        public void addNode(String nodeName) {
            if (table == null) {
                table = new Node[capacity];
            }
            // 使用节点名称定位
            int index = calcIndex(nodeName);
            int j = index;
            // 冲突，已存在节点，这里通过简单的顺移（开放寻址法）来找到合适的节点位置
            while (table[j] != null) {
                j++;
                if (j >= capacity) {
                    j = 0;
                }
                if (j == index) {
                    throw new RuntimeException("当前环已满，无法再继续添加节点！");
                }
            }
            table[j] = new Node(j, false, nodeName);
            if (needVirtual) {
                // 增加虚拟节点，减少数据倾斜。此处默认只映射一个虚拟节点，可以扩展
                int virtualIndex = calcIndex(nodeName + "-virtual");
                int k = virtualIndex;
                while (table[k] != null) {
                    k++;
                    if (k >= capacity) {
                        k = 0;
                    }
                    if (k == virtualIndex) {
                        throw new RuntimeException("当前环已满，无法再继续添加虚拟节点！");
                    }
                }
                table[k] = new Node(k, true, nodeName + "-virtual");
                // 指向实际可存储数据的节点
                table[k].actualNode = table[j];
            }
        }

        public void addData(String data) {
            int i = calcIndex(data);
            Node node;
            int j = i;
            // 找到一个可用节点。如果当前index上没有节点，则需要向后移动直至遇到一个可用节点，将数据添加到该节点
            while ((node = table[j]) == null || node.status == 0) {
                j++;
                if (j >= capacity) {
                    j = 0;
                }
                if (j == i) {
                    break;
                }
            }
            if (node == null) {
                throw new RuntimeException("找不到可用节点");
            }
            if (node.isVirtual) {
                node.actualNode.dataList.add(data);
            } else {
                node.dataList.add(data);
            }
        }

        public int calcIndex1(String data) {
            // 借用hashmap的hash算法
            int h;
            int hash = (data == null) ? 0 : (h = data.hashCode()) ^ (h >>> 16);
            return (capacity - 1) & hash;
        }
    }

    static class Node {
        int index;
        int status; // 节点状态，0表示节点不可用，1表示节点可用
        boolean isVirtual;
        Node actualNode;
        String name;
        List<String> dataList;

        public Node(int index, boolean isVirtual, String name) {
            this.index = index;
            this.isVirtual = isVirtual;
            this.name = name;
            this.status = 1;
            if (!isVirtual) {
                this.dataList = new ArrayList<>();
            }
        }
    }

}
```

上面示例代码执行一次的输出结果如下，可以看出数据很均匀，因为节点刚好占满整个环：

```tex
virtualNodeList: 
node2-virtual, index=0
node3-virtual, index=4
node4-virtual, index=6
node1-virtual, index=7

actualNodeList: 
node4, index=1, size=45222
node2, index=2, size=45153
node3, index=3, size=44864
node1, index=5, size=44761
```

如果把上面代码中**环的容量扩展为16，节点数量不变**，运行一次输出结果如下，可以看到出现了比较明显的数据倾斜：

```tex
virtualNodeList: 
node2-virtual, index=0
node1-virtual, index=7
node4-virtual, index=8
node3-virtual, index=12

actualNodeList: 
node4, index=1, size=22463
node2, index=2, size=45055
node3, index=3, size=56216
node1, index=13, size=56266
```

上述示例代码中的Hash算法采用的是HashMap中的实现，此算法有几个点值得深究：

- 为何数组长度必须是 **2^n**？
- 为何要右移 **16** 位？
- 为何要与自己做 **按位异或** 计算？
- 为何使用 **&** 取模而不是 **%** 取余？如 a % b = x，当除数b=2^n时，可以使用 & 代替 % 取余，因为 % 运算符在计算机中，是使用除法运算实现的，而除法运算的效率比位运算低很多。

可以参考这两篇文章的一些解答：

- [HashMap中的hash算法总结](https://blog.csdn.net/a314774167/article/details/100110216)

- [hash-方法的原理](https://javabetter.cn/collection/hashmap.html#_01%E3%80%81hash-%E6%96%B9%E6%B3%95%E7%9A%84%E5%8E%9F%E7%90%86)
