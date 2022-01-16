# 堆
堆的定义：n个元素的序列{k1,k2,ki,…,kn}当且仅当满足下关系时，称之为堆。如图1所示
$$(k_i \leq k_{2i} 且 k_i \leq k_{2i+1}) 或者 (k_i \geq k_{2i} 且 k_i \geq k_{2i+1})$$
根据堆的定义则有如下两个性质
- 堆中的某个节点的值总是**不大于**或者**不小于**其父节点的值
- 堆总是一颗**完全二叉树**

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
  
根据字节点和父节点的大小关系，堆又分为**大顶堆**和**小顶堆**

- **大顶堆：** 父节点的值总是大于等于子节点的值的堆称之为大顶堆
- **小顶堆：** 父节点的值总是小于等于子节点的值的堆称之为小顶堆

