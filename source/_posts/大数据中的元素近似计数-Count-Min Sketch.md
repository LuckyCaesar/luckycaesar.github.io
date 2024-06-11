---
title: 大数据中的元素近似计数-Count-Min Sketch
tags: [算法，Count-Min Sketch]
index_img: /img/Count-Min-Sketch.jpg
date: 2024-06-09 22:00:00
---

Count-Min Sketch是一种在大量数据下仍能实时且高效的近似对元素计数的算法。关于Count-Min Sketch和Data Sketching的一些了解学习可以看看这两篇文章：

- [Count-Min Sketch](https://zhuanlan.zhihu.com/p/84688298)
- [Data Sketching笔记](https://longaspire.github.io/blog/Data%20Sketching%E7%AC%94%E8%AE%B0/)

以下是一个简单的Java实现：

```java
public class CountMinSketchAlgorithm {
    
    private static final List<String> DATA_LIST = Arrays.asList(
            "a", "b", "x", "y", "a", "a", "x", "z", "a", "e", "x", "h",
            "y", "b", "x", "y", "a", "a", "x", "z", "a", "e", "x", "h",
            "y", "b", "c", "x", "y", "a", "a", "x", "z", "a", "e", "x");

    public static void main(String[] args) {
        // 根据数学模型公式来选择二维数组的合理大小（宽 = 数组长度，高 = hash函数个数）
        int wight = 256;
        int height = 4;
        int[][] array = new int[wight][height];
        // 维护一个记录频次topN（这里假定为top3）元素的集合
        LinkedList<DataWrapper> countList = new LinkedList<>();
        for (String word : DATA_LIST) {
            int count = countElement(word, wight, array);
            DataWrapper wrapper = new DataWrapper(word, count);
            countList.remove(wrapper);
            countList.push(wrapper);
            countList.sort(Comparator.comparingInt(o -> o.count));
            if (countList.size() > 3) {
                countList.pop();
            }
        }
        countList.forEach(item -> System.out.print(item.key + ":" + item.count + " "));// y:5 x:9 a:10 
    }

    private static int countElement(String e, int wight, int[][] array) {
        int h1 = hash1(e, wight);
        array[h1][0]++;
        int h2 = hash2(e, wight);
        array[h2][1]++;
        int h3 = hash3(e, wight);
        array[h3][2]++;
        int h4 = hash4(e, wight);
        array[h4][3]++;
        return Math.min(Math.min(array[h1][0], array[h2][1]), Math.min(array[h3][2], array[h4][3]));
    }

    private static int hash1(String data, int length) {
        // hashmap的hash算法
        int h;
        int hash = (data == null) ? 0 : (h = data.hashCode()) ^ (h >>> 16);
        return (length - 1) & hash;
    }

    private static int hash2(String data, int length) {
        // 乘法hash，乘数固定，类似String的hashCode
        int hash = 0;
        int i;
        for (i = 0; i < data.length(); ++i) {
            hash = 33 * hash + data.charAt(i);
        }
        return (length - 1) & hash;
    }

    private static int hash3(String data, int length) {
        // 乘法hash，乘数不固定
        int hash = 0;
        int a = 37;
        int i;
        for (i = 0; i < data.length(); ++i) {
            hash = a * hash + data.charAt(i);
            a = a * (i + 1);
        }
        return (length - 1) & hash;
    }

    private static int hash4(String data, int length) {
        // 位运算hash
        int hash, i;
        for (hash = data.length(), i = 0; i < data.length(); ++i) {
            hash = (hash << 4) ^ (hash >> 28) ^ data.charAt(i);
        }
        return (length - 1) & hash;
    }

    static class DataWrapper {
        String key;
        int count;

        public DataWrapper(String key, int count) {
            this.key = key;
            this.count = count;
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) {
                return true;
            }
            if (o == null || getClass() != o.getClass()) {
                return false;
            }
            DataWrapper wrapper = (DataWrapper) o;
            return Objects.equals(key, wrapper.key);
        }

        @Override
        public int hashCode() {
            return Objects.hash(key);
        }
    }

}
```

