
# 为什么使用二叉树

因为它结合了有序数组和链表的优点：查询和有序数组一样快，插入/删除和链表一样快。

树本质上是图的特例。

# 二叉树


先通过下图了解下树的结构及常用术语：

![树结构](../../Resource/data_structures_07_1.png)

二叉树：如果树中每个节点最多只能有两个子节点，这样的树就称之为**二叉树**，且其左节点的值小于该节点，右节点的值大于该节点。

如果树中的大部分的节点都在根的一边，我们称这样的树为**不平衡树**
![树结构](../../Resource/data_structures_07_2.png)

之所有造成不平衡的情况，是因为大部分(所有)的节点都大于或小于根节点。

我们后续在![红黑树](data_structures_08.md)一章探讨解决树不平衡的问题。

# 树操作

直接看实现吧
```java
public class Tree {

    public static class Node {
        int value;
        Node left;
        Node right;
    }

    private Node root;

    public Node find(int value) {
        Node current = root;

        while (current.value != value) {
            if (current.value < value) {
                current = current.right;
            } else {
                current = current.left;
            }

            if (current == null) {
                return null;
            }
        }
        return current;
    }

    public void insert(int value) {
        Node newNode = new Node();
        newNode.value = value;

        if (root == null) {
            root = newNode;
        } else {
            Node current = root;
            Node parent;

            while (true) {
                parent = current;

                if (value < current.value) {
                    current = current.left;
                    if (current == null) {
                        parent.left = newNode;
                        return;
                    }
                } else {
                    current = current.right;
                    if (current == null) {
                        parent.right = newNode;
                        return;
                    }
                }
            }

        }
    }

    public void walk() {
        preorder(root);
        inorder(root);
        postorder(root);
    }


    //前序遍历
    public void preorder(Node node) {
        if (node == null)
            return;

        System.out.println("node--->" + node.value);
        preorder(node.left);
        preorder(node.right);
    }

    //中序遍历
    public void inorder(Node node) {
        if (node == null)
            return;

        preorder(node.left);
        System.out.println("node--->" + node.value);
        preorder(node.right);
    }

    //后序遍历
    public void postorder(Node node) {
        if (node == null)
            return;

        preorder(node.left);
        preorder(node.right);
        System.out.println("node--->" + node.value);
    }

    public Node findMax() {
        Node currnet = root;
        Node parent = currnet;

        while (currnet != null) {
            parent = currnet;
            currnet = currnet.right;
        }
        return parent;
    }

    public Node findMin() {
        Node currnet = root;
        Node parent = currnet;

        while (currnet != null) {
            parent = currnet;
            currnet = currnet.left;
        }
        return currnet;
    }
}

```

# 删除

删除操作是二叉树中最复杂的操作。

主要流程和之前的`find`和`insert`操作是一样的：即找到要删除的节点

找到之后有三种情况：
1. 该节点是叶子节点
2. 该节点有一个子节点
3. 该节点有两个字节点


## 1.该节点是叶子节点

示意图如下:
![树结构](../../Resource/data_structures_07_3.png)

```java
 public boolean delete(int value) {
        //1.找到要删除的节点，要操作 删除节点的 父节点，将父节点的left/right置为null
        Node current = root;
        //要删除的节点是否父节点的左节点
        boolean isLeft = true;
        Node parent = current;

        while (current.value != value) {
            parent = current;
            if (value < current.value) {
                current = current.left;
                isLeft = true;
            } else if (value > current.value) {
                current = current.right;
                isLeft = false;
            }
            if (current == null) {
                return false;//未找到
            }
        }
        //2. 删除
        if (current.left == null && current.right == null) {

            if (root == current) {
                root = null;
            }
            if (isLeft) {
                parent.left = null;
            } else {
                parent.right = null;
            }
            current = null;

            return true;
        }
        else
 {

 }

        return true;
    }
```
## 2.该节点有一个子节点

这种情况也不复杂，我们称要删除的节点为`deleteNode`


只需要把`deleteNode`的父节点的`left`/`right`指向`deleteNode`的唯一子节点
示意图如下:
![树结构](../../Resource/data_structures_07_4.png)

```java
 if (current.left == null) {
    //2.2 有一个right子节点
    if (root == current) {
        root = current.right;
    } else {
        if (isLeft) {
            parent.left = current.right;
        } else {
            parent.right = current.right;
        }
    }

} else if (current.right == null) {
    //2.2 有一个left子节点
    if (root == current) {
        root = current.left;
    } else {
        if (isLeft) {
            parent.left = current.left;
        } else {
            parent.right = current.left;
        }
    }
```

## 3.该节点有两个子节点

下图 是一个错误的代替方法：用右子树替代删除节点的示意图：
![删除-右子树取代](../../Resource/data_structures_07_5.png)

关键点在于：

**对每一个节点来说，比该节点的值 次高 的是它的中序后继节点**

所以有两个节点的处理方式也就显而易见了：用它的中序后继节点代替之。
![删除-中序后继](../../Resource/data_structures_07_6.png)


如何定位中序后继节点呢？

首先找到`deleteNode`的右节点，因为右节点树中所有的节点都大于`deleteNode`，所以只需要找到 **右节点树** 中的最小值即可

如果右节点没有子左节点，则右节点本身就是后继节点；否则递归找到右节点的子左节点

示意图如下：
![删除-中序后继](../../Resource/data_structures_07_7.png)


**TODO:**

比较取巧的办法是在`Node`中添加一个`isDelete`属性，删除时候直接将该属性置为`true`

# 用数组表示树
还有一种完全不同的方法表示树：数组

用数组的方法表示树，节点保存在数组中，节点间不是通过引用相连。

其中下标为0的是根，下标为1的是根的左子节点，一次类推
![数组表示树](../../Resource/data_structures_07_8.png)


设置节点索引值为`index`,则左子节点是`2*index+1`,右子节点是`2*index+2`，父节点是`(index-1)/2`

大都数情况下，数组表示树不是很有效率，不满的节点和删除的节点都在数组中留下了洞，浪费了存储空间。更坏的是删除节点时需要移动子树的化，子树每个节点都要移到数组中新的位置，很费时。

# 重复值

目前 插入时候都是插入到值相同的右子节点。

取巧的做法是禁止出现重复关键字


# Huffman编码

哈夫曼编码是采用二叉树的方式来压缩数据的。右`David Huffman`在1952年发现该方法。