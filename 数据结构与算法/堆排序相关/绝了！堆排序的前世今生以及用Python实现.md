## 堆排序前传

### 树与二叉树

#### 树的简单定义

树是一种数据结构，比如：目录结构。  
树是一种可以递归定义的数据结构。  
树是由于 n 个节点组成的集合：
- 如果 n=0，则那就是一棵空树；
- 如果 n>0，那就是存在一个节点作为树的根几点，其它节点可以分为 m 个集合，每个集合本身又是一棵树。如下图所示：
![tree.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51b0742f22394a55b75e235a1f8c25d0~tplv-k3u1fbpfcp-watermark.image)

#### 树的一些概念

- 根节点：树的第一个节点，上图中就是 A 节点；
- 叶子节点：树中不能再分叉的节点，上图中 B、C、H、I、..等；
- 树的深度：最深有几层，就是树的高度，上图中树的深度为 4；
- 树的度：整棵树中哪个节点分叉最多，即为树的度，上图中树的度为 6，A 节点的分叉最多；
- 父节点/孩子节点：上层节点即为下层节点的父节点，下层节点即为上层节点的子节点。

### 二叉树

### 二叉树定义

- 度不超过 2 的树
- 每个节点最多有两个孩子节点
- 两个孩子节点被区分为左孩子节点和右孩子节点

如下图所示：
![二叉树.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd8a3c1604fe4e67b0fefb7536624af1~tplv-k3u1fbpfcp-watermark.image)

### 满二叉树和完全二叉树

#### 满二叉树

一个二叉树，如果每一层的节点数都达到最大值，则这个二叉树就是满二叉树。

#### 完全二叉树

叶子节点只能出现在最下层和次下层，并且最下面一层的节点都集中在该层坐左边的若干位置的二叉树。

![完全二叉树满二叉树.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/201039265a4841a5a83e649a53b69e55~tplv-k3u1fbpfcp-watermark.image)

其中：(a)为满二叉树，(b)为完全二叉树，(c)和(d)为非完全二叉树。

### 二叉树存储方式（表达方式）

- 链式存储方式
- **顺序存储方式（重点讲）**

#### 顺序存储方式

简单来树就是用列表来存，如下图所示：
![二叉树存储方式.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68265565771345c393f6396b4e8e0f18~tplv-k3u1fbpfcp-watermark.image)

父节点和左孩子节点的编号下标的关系：
- 0->1, 1->3, 2->5, 3->7, 4->9
- i->2i+1

父节点和右孩子节点的编号下标的关系：
- 0->2, 1->4, 2->6, 3->8, 4->10
- i->2i+2

## 堆排序

### 什么是堆

一种特殊的完全二叉树结构

- 大根堆（大顶堆）：一棵完全二叉树，满足任意一节点都比其孩子节点都大
- 小根堆（小顶堆）：一棵完全二叉树，满足任意一节点都比其孩子节点都小

大根堆和小根堆如下图所示：

![大小根堆.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b3a6dbe531f40e6979f1cfdd30bc279~tplv-k3u1fbpfcp-watermark.image)

### 堆的向下调整性质

假设跟节点的左右子树都是堆，但根节点不满足堆的性质，则可以通过一次向下调整来将其变成一个堆。

### 堆排序-构造堆

#### 构造堆过程：农村包围城市

1. 从树的最后一级有孩子节点开始看起（即最后一个非叶子节点），每次先调整小的子树，再调整大的子树，最后调整整棵树。

其过程如下：
1）从树的最后一级有孩子节点开始看起
![构造堆过程00.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dfc6a061b7df4dc69b528012f50834a2~tplv-k3u1fbpfcp-watermark.image)

2）对子树作调整（3和5交换，可以看作换村长）
![构造堆过程01.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ffef09a56df74d479754578c7570ad03~tplv-k3u1fbpfcp-watermark.image)

3）继续对另一颗子树调整，可以看到该子树符合大根堆，无需再做调整
![构造堆过程02.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/88d73978794d47efa0d03eedf8fe00c4~tplv-k3u1fbpfcp-watermark.image)

4）看完村级别的，继续看县级的，并做调整
![构造堆过程03.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/231319d55d164b08b27f0344e8a6bfb2~tplv-k3u1fbpfcp-watermark.image)
![构造堆过程04.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a7570f2ee7a4dc6957d99429b4c8d25~tplv-k3u1fbpfcp-watermark.image)

5）最后做进一步的调整
![构造堆过程05.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb09b63468754bc5b755429203e9d53a~tplv-k3u1fbpfcp-watermark.image)

可以看到，通过以上过程，一颗大根堆就已经构建好了。

接下来，进行挨个出数（出数的过程会进行向下调整），即可拿到排序好的这些数，即我们所说的**堆排序**。

### 堆排序-挨个出数

#### 挨个出数

对下面一颗树（上面构造好的一棵大根堆树）进行挨个出数，树如下：
![构造堆过程1.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/577b92a654714d6ebdd7b092d7e9f6f8~tplv-k3u1fbpfcp-watermark.image)

挨个出数的过程如下：

1）出数 9，把做最后一个节点 3 放在根节点
![构造堆过程2.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/254d23ac887b49149bc31b9c4652229b~tplv-k3u1fbpfcp-watermark.image)

2）然后进行向下调整
![构造堆过程3.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f03a50e8c9d045c48256ca6ec808a1d6~tplv-k3u1fbpfcp-watermark.image)

3）出数 8，把最后一个节点 3 放在根节点
![构造堆过程4.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0364c72b1aca4ea9946075682a02d333~tplv-k3u1fbpfcp-watermark.image)

4）接着再进行向下调整
![构造堆过程5.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c81852f2f4f643be85041ee0b02cdcbf~tplv-k3u1fbpfcp-watermark.image)

5）以此类推，直到这棵树为空。

最后出数的即为有序的数，也就是排好序的数，这就是**堆排序**。

总的来说，堆排序过程如下：
1. 建立堆
2. 得到堆顶元素，为最大元素
3. 去掉堆顶元素，将堆最后一个元素放到堆顶，此时可通过一次调整重新使堆有序
4. 堆顶元素为第二大元素
5. 重复步骤3，直到堆变空。

### 堆排序实现

```python

#!/usr/bin/env python
# -*- encoding: utf-8 -*-
'''
@Author  :   Scoefield 
@File    :   heap_sort.py
@Time    :   2021/05/23 20:26:35
'''

from collections import deque
import random

def swap_param(L, i, j):
    # 交换父子节点的值, 让父节点大于两个子节点
    L[i], L[j] = L[j], L[i]
    return L

# 构造大根堆
def heap_adjust(L, start, end):
    # 将父节点的值赋值给temp
    temp = L[start]
    # 将父节点的索引赋值给i
    i = start
    # 左子节点的索引
    j = 2 * i
    # 当j在L的长度范围之内时
    while j <= end:
        # 如果子节点在范围内并且左子节点小于右子节点
        if (j < end) and (L[j] < L[j + 1]):
            # 将j切换成右子节点
            j += 1
        # 如果父节点的值比子节点中比较的节点小
        if temp < L[j]:
            # 将子节点的值赋值给父节点
            L[i] = L[j]
            # 将子节点看成父节点
            i = j
            # 将子节点的子节点看成子节点
            j = 2 * i
        else:
            break
    L[i] = temp

# 堆排序
def heap_sort(L):
    # 数组的元素个数中多了辅助位置, 需要减去1
    L_length = len(L) - 1
    # 完全二叉树的节点按照层次并按从左到右的顺序从0开始编号,那么第n个节点的父节点为n//2
    first_sort_count = L_length // 2
    for i in range(first_sort_count):
        # 构造大根堆
        heap_adjust(L, first_sort_count - i, L_length)
    for i in range(L_length - 1):
        # 将堆顶元素与堆最右方下方的元素交换
        L = swap_param(L, 1, L_length - i)
        # 构造大根堆
        heap_adjust(L, 1, L_length - i - 1)
    return [L[i] for i in range(1, len(L))]

def main():
    # 将列表生成链表
    # L = deque([17 , 13, 40 , 22,  31, 14,  33, 56,  24, 19, 10, 41, 51, 42, 26])
    # 因为堆得节点从1开始, 引入辅助位置, 使数组的下标从1开始
    # L.appendleft(0)

    # 随机生成列表
    li = [i for i in range(20)]
    random.shuffle(li)

    # 堆排序并打印排序结果
    res = heap_sort(li)
    print(res)

if __name__ == '__main__':
    main()
```

**堆排序的时间复杂度：O(nlogn)**。
