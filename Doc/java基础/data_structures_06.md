# 一. 归并排序

原本归并排序是在 递归 这一章介绍的，但是讲解快速排序时候使用到了归并，所以调整下，这里先介绍下归并排序。

## 1. 归并算法
归并算法的中心是合并两个已经有序的数组，生成第三个数组。

示意图如下：

![归并算法](../../Resource/data_structures_06_1.png)

代码如下：
```java

    void mergeSort() {
        int[] a = {23, 47, 81, 95};
        int[] b = {7, 14, 39, 55, 62, 74};

        int[] c = new int[a.length + b.length];

        int aIndex = 0;
        int bIndex = 0;
        int cIndex = 0;

        while (aIndex < a.length - 1 && bIndex < b.length - 1) {
            if (a[aIndex] < b[bIndex]) {
                c[cIndex++] = a[aIndex++];
            } else {
                c[cIndex++] = b[bIndex++];
            }
        }

        while (aIndex < a.length - 1) {
            c[cIndex++] = a[aIndex++];
        }

        while (bIndex < b.length - 1) {
            c[cIndex++] = b[bIndex++];
        }

    }
```

## 1. 归并排序


# 二. 希尔排序

# 三. 划分

# 四. 快速排序

# 五. 基数排序


