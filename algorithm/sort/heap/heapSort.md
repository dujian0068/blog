## 堆
### 堆的定义
任意一个子节点总是大于等于或者小于等于父节点的完全二叉树称之为堆，根据字节点和父节点的大小关系，堆又分为**大顶堆**和**小顶堆**

- **大顶堆：** 父节点的值总是大于等于子节点的值的堆称之为大顶堆，大顶堆的最大值总是在堆顶
- **小顶堆：** 父节点的值总是小于等于子节点的值的堆称之为小顶堆，小顶堆的最小值总是在堆顶
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);
    "
    src="./pic/大顶堆和小顶堆.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">图1：大顶堆（图左）和小顶堆（图右）</div>
</center>

### 堆的存储结构
堆本质是一棵完全二叉树，则完全二叉树的所有性质都适合于堆。
对于二叉树我们一般定义节点的数据结构如下：
```
public class Node<T> {
    public T data;
    public Node left;   //左孩子节点
    public Node right;  //右孩子节点  
    public Node parent; //父节点
}
```
但是对于完全二叉树，将其按照层序遍历得到包含二叉树中所有元素的序列(${k_1,k_2,...,k_i,...k_n}$)，总是有给定一个节点$k_i$,那么其**左孩子**节点为$k_{2i+1}$，**右孩子**为$k_{2i+2}$，其**父节点**为$k_{(i-1)/2}$,那么就可以完全省略掉树节点中左孩子、右孩子以及父节点的相关定义，使用一维数组来存储堆。图1所示的堆使用数组表示后如图2所示

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);
    "
    src="./pic/大顶堆和小顶堆的数组表示.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">图2：大顶堆（图左）和小顶堆（图右）</div>
</center>

## 堆排序
现有一数组为[23,34,5,21,13,45,34,78,8],其完全二叉树表示如图3，以此数组为例进行堆的构建

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);
    "
    src="./pic/原始数组.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">图3：原始数组</div>
</center>

### 堆的构建
#### 大顶堆的构建

