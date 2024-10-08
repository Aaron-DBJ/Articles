# 概念

B树不要和二叉树混淆，在[计算机科学中](https://en.wikipedia.org/wiki/Computer_science "计算机科学")，**B树**是一种自平衡[树数据结构](https://en.wikipedia.org/wiki/Tree_data_structure "树数据结构")，它维护有序数据并允许以[对数时间](https://en.wikipedia.org/wiki/Logarithmic_time "对数时间")进行搜索，顺序访问，插入和删除。B树是[二叉搜索树](https://en.wikipedia.org/wiki/Binary_search_tree "二进制搜索树")的一般化，因为节点可以有两个以上的子节点。[[1]](https://en.wikipedia.org/wiki/B-tree#cite_note-Comer-1)与其他[自平衡二进制搜索树不同](https://en.wikipedia.org/wiki/Self-balancing_binary_search_tree "自平衡二叉搜索树")，B树非常适合读取和写入相对较大的数据块（如光盘）的存储系统。它通常用于[数据库](https://en.wikipedia.org/wiki/Database "数据库")和[文件系统](https://en.wikipedia.org/wiki/File_system "文件系统")。

# 定义

B树是一种平衡的多分树，通常我们说m阶的B树，它必须满足如下条件： 

- 每个节点最多只有m个子节点。
- 每个非叶子节点（除了根）具有至少⌈ m/2⌉子节点，含有ceil(m/2)-1到m-1个元素。
- 如果根不是叶节点，则根至少有两个子节点。
- 具有*k*个子节点的非叶节点包含*k* -1个键。
- 所有叶子都出现在同一水平，没有任何信息（高度一致）。

## 阶

B树中一个节点的子节点数目的最大值，用m表示。

例如有以下B树，所有节点中，节点【13,16,19】拥有的子节点数目最多，四个子节点（灰色节点），所以可以定义上面的图片为4阶B树

![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240927112116.png)

**什么是根节点 ？**

节点【10】即为根节点。

特征：根节点拥有的子节点数量的上限和内部节点相同，如果根节点不是树中唯一节点的话，至少有俩个子节点（不然就变成单支了）。在m阶B树中（根节点非树中唯一节点），那么根结点有关系式2<= M <=m，M为子节点数量；包含的元素数量 1<= K <=m-1,K为元素数量。

**什么是内部节点 ？**

节点【13,16,19】、节点【3,6】都为内部节点。

特征：内部节点是除叶子节点和根节点之外的所有节点，拥有父节点和子节点。假定m阶B树的内部节点的子节点数量为M，则一定要符合（m/2）<=  M <=m关系式，包含元素数量M-1；包含的元素数量 （m/2）-1<= K <=m-1,K为元素数量。m/2向上取整。

**什么是叶子节点？**

节点【1,2】、节点【11,12】等最后一层都为叶子节点，叶子节点对元素的数量有相同的限制，但是没有子节点，也没有指向子节点的指针。

特征：在m阶B树中叶子节点的元素符合（m/2）-1<= K <=m-1。
