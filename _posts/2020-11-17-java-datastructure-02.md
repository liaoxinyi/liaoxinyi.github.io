---
layout:     post
title:      "东拉西扯数据结构-02"
subtitle:   "二叉树、二叉查找树、平衡二叉树、红黑树、B-Tree、B+Tree、B\*Tree"
date:       2020-11-17
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-bkg03.jpg"
tags:
    - Java
    - 数据结构
---
> 《我是攻城狮》+《Java极客技术》+《数据结构与算法分析 java语言描述》（原书第3版）-读书笔记

### 树的一些概念
树是一种无向图，**其中任意两个节点间存在唯一一条路径**。树常常用一种递归的方式来定义，它是一种数据结构，要么为空，要么具有一个值并且有零个或多个孩子，而每个孩子又是树  
- 根（root）、边（edge）、子（child）、父（parent）
上面这几个都好理解，顾名思义  
- 树叶（leaf）：没有儿子的节点  
- 兄弟（brother）：具有相同父亲的节点  
- 路径（path）和长（length）：从节点n1到nk的路径为n1、n2、...nk的一个序列，该路径上边的条数为长
- 深度（depth）  
    - 从根到ni的唯一路径的长  
    - 根的深度为0  
- 高（height）  
    - 从ni到一片树叶的最长路径的长  
    - 所有树叶的高都是0  
    - 一棵树的高度等于它根的高  
- 树的层：树的深度+1，根节点的层为1  
- 树的实现  
在表的数据结构中，有数组、链表、栈和队列  

### 树的遍历
##### 基本思想
普通的线性数据结构如链表，数组等，遍历方式单一，都是从头到尾遍历就行，但树这种数据结构却不一样，从一个节点出发，下一个节点却有可能遇到多个分支路径，所以为了遍历树的全部节点，需要借助一个临时容器，通常是栈或者队列这种数据结构，来存储当遇到多个分叉路径时的，存暂时没走的其他路径，等走过的路径遍历完之后，再继续返回到原来没走的路径进行遍历，这一点不论在递归中的遍历还是迭代中的遍历中其实都是一样的，只不过递归方法的栈是隐式的，而自己迭代遍历的栈需要显式的声明  
##### 深度优先遍历（Depth-First-Search=>DFS）  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/tree-iterator-depth.jpg)

- 先序遍历(preorder traversal)  
先根节点，然后左子树，最后右子树    
- 后序遍历(postorder traversal)  
先左子树，然后右子树，最后根节点  
- 中序遍历（inorder traversal）  
先左子树，然后根节点，最后右子树  

可以理解为，根据以什么顺序遍历根节点来命名这三种深度优先遍历方式，接下来用图解释一下采用额外的栈来进行树的遍历的原理：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/tree-iterator-depth-ex1.jpg)  
以中序遍历为例，先左子树，然后根节点，最后是右子树。按照这个顺序，首先把这三部分数据，放在一个栈里，如下图所示：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/tree-iterator-depth-ex2.jpg)  
然后遍历的过程就是出栈的过程，但如果栈里的结构仍然是一颗子树，就仍然需要按照中序遍历的规则，来继续拆分数据入栈，直到这颗子树，被分解成一个个叶子节点。在上图的栈里，可以发现栈顶部分仍然是一颗子树，所以需要继续拆分，按照先左，再根，最后右来拆分，继续入栈，最后图示如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/tree-iterator-depth-ex3.jpg)  
此时对应的遍历结果即为：5 -> 12 -> 6 -> 1 -> 9

##### 广度优先遍历（Breadth-First-Search=>BFS）  
遍历的过程就是对每一层节点依次访问，访问完一层进入下一层，而且每个节点只能访问一次，可以通过使用队列来完成遍历    
- 层级遍历（levelorder traversal）  
在一棵树中把节点从左往右，一层一层的从上往下遍历输出，大致流程如下：  
1. 首先将根节点放入队列中  
2. 当队列为非空时，循环执行步骤3到步骤5，否则执行6  
3. 出队列取得一个结点，访问该结点  
4. 若该结点的左子树为非空，则将该结点的左子树入队列  
5. 若该结点的右子树为非空，则将该结点的右子树入队列  
6. 结束  

### 二叉树（binary tree）
二叉树就是一棵树，其中**每个节点最多两个子节点**  
- 平均深度：O（N^0.5）  
- 最坏情况下深度：N-1  
- 既有链表的好处，也有数组的好处，是两者的优化方案

##### 结构优势
- 二叉树的每个节点其实都可以看做是个索引，所以二叉树这种数据结构很适合用来查找  
- 用于维持相对顺序  

### 二叉查找树（Binary Search Tree，BST）
##### 定义
- 若任意节点的左子树不空，则左子树上所有节点的值均小于它的根节点的值
- 若任意节点的右子树不空，则右子树上所有节点的值均大于或等于它的根节点的值  
- 任意节点的左、右子树也分别为二叉查找树
- 可以是空树

##### 结构优势
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/bst-image-wiki.jpg)

- 又称为二叉排序树、二叉查找树
- 平均深度：O(logN)  
- 插入和删除等效率比较高，时间复杂度为：O(logN)  
- 二叉查找树通常情况下采用链式存储  

可以发现，最差的情况降低到了O(n)的情况，这就是为啥会发展出平衡二叉树的原因  

##### 插入操作
因为二叉查找树的深度不深，所以往往通过递归的方法就能完成  
##### 查找操作
查找的思路有点像二分查找，我们知道根节点左子树的值都比根节点值小，而右子树的值都比根节点的值要大，以此递归调用，可以很容易找到  
##### 删除操作  
- 判断根节点如果为空，直接返回  
- 判断根节点不为空，分情况考虑：  
1. 被删除的节点，右子树为空  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/bst-delete-01.jpg)  
这种场景，只需要将被删除元素的左子树的父节点移动到被删除元素的父节点，然后将被删除元素移除即可  
2. 被删除的节点，左子树为空  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/bst-delete-02.jpg)  
这种场景，与上面类似，只需要将被删除元素的右子树的父节点移动到被删除元素的父节点，然后将被删除元素移除即可  
3. 被删除的节点，左、右子树不为空  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/bst-delete-03.jpg)  
这种场景，稍微复杂一点，先定位到要删除的目标元素，**根据左子节点内容一定小于当前节点内容特点，找到目标元素的左子树，通过递归遍历找到目标元素的左子树的右子树，找到最末端的元素之后，进行与目标元素进行替换，最后移除最末端元素**  

**换句话说：此时就是找到删除节点的右节点的最末端左子树，进行替换并删除**

总体来说，删除操作并不是一次简单的查找就可以完成，甚至删除会使得整个二叉树变得非常不平衡，所以如果删除次数不多，完全可以采用懒惰删除，即当节点被删除时，仅做一个标记，表明它被删除了，而不是真的从树中移除，这样虽然表面上浪费了一点空间，但是如果后面又要插入该元素值，则为新元素分配内存的开销就免了  
### 平衡二叉树（AVL tree：Adelson-Velskii和Landis）
有了二叉查找树，其实在查找的时候已经很快了，为啥还要有这个平衡二叉树呢？因为二叉查找树可能存在一种情况，所有的子节点都在根节点的右边（排成了一长串），退化成链表，此时时间复杂度为 O(n)，所以AVL树应运而生  
##### 定义
平衡二叉树是一种结构平衡的二叉查找树，所以关键点在于平衡是怎么判断的：    
- 它的左右两个子树的高度差的绝对值不超过1  
这里涉及到一个概念：**平衡因子（左子树的高度减去右子树的高度）**，所以**平衡二叉树的平衡因子的取值只可能为0，-1，1**  
- 左右两个子树都是一颗平衡二叉树  

##### 结构优势
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/avl-image-wiki.jpg)

##### 插入操作
当插入元素时，有可能破坏平衡二叉树的结构，所以当加入节点后，需要沿着节点向上维护平衡性  
- 最小失衡子树  
在新插入的结点向上查找，以第一个平衡因子的绝对值超过1的结点为根的子树称为最小不平衡子树。平衡二叉树的失衡调整主要是通过旋转最小失衡子树来实现，**左旋与右旋**  
- 左旋RR  
插入的元素在不平衡的节点的右子树的右子树  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/avl-add-rr.jpg)  
1. 节点的右孩子替代此节点位置：节点 66 的右孩子是节点 75 ，将节点 75 代替节点 66 的位置  
2. 右孩子的左子树变为该节点的右子树：节点 75 的左子树为节点 70，将节点 70 挪到节点 66 的右子树位置  
3. 节点本身变为右孩子的左子树：节点 66 变为了节点 75 的左子树  

- 右旋LL  
插入的元素在不平衡的节点的左子树的左子树  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/avl-add-ll.jpg)    
1. 节点的左孩子替代此节点：  
2. 节点的左孩子的右子树变为节点的左子树：  
3. 节点本身变为左孩子的右子树：  

- 右旋LR  
插入的元素在不平衡的节点的左子树的右子树  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/avl-add-lr.jpg)  
先进行左旋转，将树结构调整为LL结构，然后再进行右旋转，实现AVL  

- RL  
插入的元素在不平衡的节点的右子树的左子树  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/avl-add-rl.jpg)  
先进行右旋转，然后再进行左旋转  

### 红黑树（Red–black tree，RBT）
虽然平衡二叉查找树是一个高度平衡的二叉树，有各种优势，但是在删除的时候，可能需要多次调整，也就是左旋转、右旋转操作，在树的深度很大的情况下，删除效率会非常低，如何提高这种效率？红黑树就产生了  
##### 定义
- 每个节点要么是黑色要么是红色  
- 根节点是黑色  
- 每个为空的叶子节点是黑色  
- 如果一个节点是红色的，则它的子节点必须是黑色的；（说明父子节点之间不能出现两个连续的红节点）  
- 从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点  

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-01.jpg)  

##### 结构优势
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/rbt-image-wiki.jpg)

- 红黑树是一个基本平衡的二叉树，在查询方面，与二叉查找树思路相同；**在插入方面，单次回溯不会超过2次旋转；在删除方面，单次回溯不会超过3次旋转！**  

- 相比于平衡二叉树，红黑树在查询、插入方面，效率差不多；**在删除方面，平衡二叉树最多需要log(n)次变化，以达到严格平衡，而红黑树最多不会超过3次变化，因此效率要高于平衡二叉树！**  

- 在实际应用中，很多语言都实现了红黑树的数据结构，比如 Java 中的 TreeMap、 TreeSet，以及 jdk1.8 中的 HashMap！  

##### 插入操作
- step1:将红黑树当作一颗二叉查找树，将节点插入树的底部  
比较好理解，红黑树其实就是二叉树的一种特殊形态的树形结构，先找到合适的位置，然后将节点插入到树上  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-02.jpg)  

- step2:默认将插入的节点着色为红色，如果是根节点，颜色着为黑色   
- step3:通过一系列的旋转或着色等操作，使之重新成为一颗红黑树  

为什么新插入的节点要设置为红色呢？因为插入之前所有根至外部节点的路径上黑色节点数目都相同，所以如果插入的节点是黑色，肯定会导致黑色节点数目不相同！而相对的插入红节点可能也会违反不能出现两个连续的红节点，如果违反条件，直接进行颜色转换或者旋转操作即可！相对将插入的节点着色为黑色，红色操作可能更简单些！因为根节点为黑色，如果是新插入的节点为根节点，直接将颜色设置为黑色！既然新插入的节点为红色，那么需要就不同的场景来分析一下，例如：  
1. **插入的节点，父亲为黑色**  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-03.jpg)  
因为新插入的节点为红色，因此不会违反任何特性，所以不需要调整  
2. **插入的节点，父亲为红色**  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-04.jpg)  
因为新插入的节点为红色，违反不能出现两个连续的红节点，因此需要进行调整！具体调整起来就需要另外细分三个情形：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-05.jpg)  
    - z的叔节点y是红色
    ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-06.jpg)  
    因为节点 z 是一个红色节点，其父节点 a 是红色的，违反了特性4，因此需要进行调整。因为其叔节点 y 是红色的，于是可以修改节点 a、节点 y 为黑色，此时节点 c 的黑高会发生变化，由原来的黑高1变成黑高2，为了节点 c 保持黑高不变，将其变成红色  
    
    此时，由于节点 c 由黑色变成了红色，如果节点 c 的父节点是红色，也会违反特性4，继续将节点 c 看成是节点 z，向上回溯调整树！（这个想想都有点麻烦啊！）  
    
    需要注意的是：对于新插入的节点 z 是节点a 的左子树的情况，操作与上述一致；对于新插入的节点 z 是节点 c 的右子树的节点的情况，操作与上述对称！

    - z的叔节点y是黑色，且z是一个左孩子
    ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-07.jpg)  
    这种情况，并不能像上面那样改变节点颜色就可以满足要求，因为如果将节点z 的父节点 a 变成了黑色， 那么树的黑高就会发生变化，必然会引起对性质5的违反。比如，此时节点y为黑色， 节点c 的右子树高度为2（忽略子树），左子树高也相同，如果简单的修改节点a 为黑色，那么节点c 的左子树的黑高会比右子树大1， 此时即使将节点c 修改为红色也于事无补！  
    
    因此，单靠颜色转变无非解决问题，需要进行旋转调整。先绕节点 a 的父节点进行右旋转，再将节点 a、节点 c 的颜色进行互换！最终结果与插入前一致！  
    
    需要注意的是：对于新插入的节点 z 是节点 c 的右子树的节点的情况，操作与上述对称！  
    
    - z的叔节点y是黑色，且z是一个右孩子
    ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-08.jpg)  
    当节点 z 是一个右孩子时，先绕节点 a 进行左旋转，之后树的形态就变成如上面介绍的情况2，再进行右旋转、颜色转变，即可实现红黑树的特性！  
    
    需要注意的是：对于新插入的节点 z 是节点 c 的右子树的节点的情况，操作与上述对称！
    
**可以得出如下结论：对插入节点后的调整所做的旋转操作不会超过2次！**

##### 删除操作
- 第一步：将红黑树当作一颗二叉查找树，将节点删除  
这里有两个个概念有点神奇：  
1. **删除节点**    
这个删除节点，某些情况下并非就是指定删除的那个节点，对于拥有左子树、右子树的节点来说，删除节点指的是被删除节点的后继节点或者前驱节点！话不多说，直接上图理解：  
2. **被删节点**  
可以就理解为指定删除的那个接口

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-09.jpg)  
如上图，传入需要删除的节点是 80，但是实际上删除节点的是85节点（85是一个叶子节点），然后将80节点内容替换成85，做了一个偷天换日的操作！  

所以有一个结论：**删除节点一定是一个单分支节点或者叶子节点** 

- 第二步：通过旋转和变色等一系列来修正该树，使之重新成为一棵红黑树  
删除操作，归根结底还是要进行一系列修正的，否则红黑树的规则很容易被打破  
1. **删除的节点为红色**  
当删除的节点为红色时，这种情况直接将删除节点移除  
为啥这么干脆，首先树中各个节点的黑高没有变化，其次删除后肯定也不会出现红红相连情况，当然删除的不可能是根节点，因为根节点是黑色的（就喜欢这样干脆的，不用额外进行什么处理）  
2. **删除的节点为黑色**  
当删除的节点为黑色时，因为删除节点的父节点失去了一个黑色子节点，这种情况会导致左右子树不平衡，因此需要进行调整，假设`x`为替换节点  
    
    - 1.0x的兄弟节点w是红色的
    ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-10.jpg)  
    因为节点 c 的左子树被删去了一个黑色节点，导致节点 c 的左子树黑高少了1，所以节点c 是不平衡的。可以对节点c 进行一次左旋，然后修改节点80和节点120 的颜色  
    
    此时，x的父亲节点c依然不平衡，节点x 的兄弟节点w 变成黑色的！  
    
    - 2.0x的兄弟节点w是黑色的，并且w的两个子节点都是黑色的  
        - 2.1x的父节点为红色  
        ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-11.jpg)  
        在删除节点后，左图节点 c 不平衡，节点c 左子树的黑高为hl+1，节点c 左子树的黑高为hr+2，左子树黑高小于右子树黑高。因此直接将节点c 修改为黑色，节点 w 修改为红色，此时的黑高又恢复如初！但是节点c 作为子节点，因为黑高减少，需要继续向上回溯调整树的黑高，此时节点c 作为新的节点x  
        
        - 2.2x的父节点为黑色  
        ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-12.jpg)  
        只需要将节点 w 修改为红色，继续向上回溯调整树的黑高，此时节点 c 作为新的节点x  
        
    - 3.0x的兄弟节点w是黑色的，并且w的右孩子是红色的  
        - 3.1x的父节点为黑色，w的左孩子是黑色的
        ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-13.jpg)  
        因为删除节点导致节点c 不平衡，对节点c 进行一次左旋转，将节点w 的右孩子颜色修改为黑色。此时节点c 已经达到平衡，同时节点w 也达到平衡，整棵树已经平衡了  
        
        - 3.2x的父节点为黑色，w的左孩子是红色的  
        ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-14.jpg)  
        同样的，对节点c 进行一次左旋转，将节点w 的右孩子颜色修改为黑色。此时节点c 已经达到平衡，同时节点w 也达到平衡，整棵树已经平衡了  
        
        - 3.3x的父节点为红色，w的左孩子是黑色的  
        ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-15.jpg)  
        对节点c 进行一次左旋转，将节点 w 的右孩子颜色修改为黑色，同时将节点80 和节点100 颜色进行互换。此时节点c 已经达到平衡，同时节点w 也达到平衡，整棵树已经平衡了  
        
        - 3.4x的父节点为红色，w的左孩子是红色的  
        ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-16.jpg)  
        同样的，对节点c 进行一次左旋转，将节点 w 的右孩子颜色修改为黑色，同时将节点80 和节点100 颜色进行互换。此时节点c 已经达到平衡，同时节点w 也达到平衡，整棵树已经平衡了  
        
    - 4x的兄弟节点w是黑色的，而且w的左孩子是红色的，w的右孩子是黑色的  
        - 4.1x的父节点为红色  
        ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-17.jpg)  
        对节点w 进行一次右旋转，将节点90和节点100 进行颜色互换，此时节点x 和节点w 的关系变成：x的兄弟节点w是黑色的，并且w的右孩子是红色的。此时按照case3.3情况进行处理即可  
        
        - 4.2x的父节点为黑色  
        ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-rbt-18.jpg)  
        同样的，对节点w 进行一次右旋转，将节点90和节点100进行颜色互换，此时节点x 和节点w 的关系变成：x的兄弟节点w是黑色的，并且w的右孩子是红色的。此时按照case3.1情况进行处理即可  
        
**此次节点x 位于节点c 的左子树，如果位于右子树，操作与之对称！**

##### 代码-RBT树的实体结构
```java
public class RBTNode<E extends Comparable<E>> {

    /**节点颜色*/
    boolean color;

    /**节点关键字*/
    E key;

    /**父节点*/
    RBTNode<E> parent;

    /**左子节点*/
    RBTNode<E> left;

    /**右子节点*/
    RBTNode<E> right;

    public RBTNode(E key, boolean color, RBTNode<E> parent, RBTNode<E> left, RBTNode<E> right) {
        this.key = key;
        this.color = color;
        this.parent = parent;
        this.left = left;
        this.right = right;
    }

    @Override
    public String toString() {
        return "RBTNode{" +
                "color=" + (color ? "red":"black") +
                ", key=" + key +
                '}';
    }
}
```

##### 代码-RBT树的插入和删除
```java
public class RBTSolution<E extends Comparable<E>> {

    /**根节点*/
    public RBTNode<E> root;

    /**
     * 颜色常量 false表示红色，true表示黑色
     */
    private static final boolean RED = false;
    private static final boolean BLACK = true;


    /*
     * 新建结点(key)，并将其插入到红黑树中
     * 参数说明：key 插入结点的键值
     */
    public void insert(E key) {
        System.out.println("插入[" + key + "]:");
        RBTNode<E> node=new RBTNode<E>(key, BLACK,null,null,null);
        // 如果新建结点失败，则返回。
        if (node != null)
            insert(node);
    }

    /*
     * 将结点插入到红黑树中
     *
     * 参数说明：
     *     node 插入的结点        // 对应《算法导论》中的node
     */
    private void insert(RBTNode<E> node) {
        int cmp;
        RBTNode<E> y = null;
        RBTNode<E> x = this.root;

        // 1. 将红黑树当作一颗二叉查找树，将节点添加到二叉查找树中。
        while (x != null) {
            y = x;
            cmp = node.key.compareTo(x.key);
            if (cmp < 0)
                x = x.left;
            else
                x = x.right;
        }

        node.parent = y;
        if (y!=null) {
            cmp = node.key.compareTo(y.key);
            if (cmp < 0)
                y.left = node;
            else
                y.right = node;
        } else {
            this.root = node;
        }

        // 2. 设置节点的颜色为红色
        node.color = RED;

        // 3. 将它重新修正为一颗二叉查找树
        insertFixUp(node);
    }

    /*
     * 红黑树插入修正函数
     *
     * 在向红黑树中插入节点之后(失去平衡)，再调用该函数；
     * 目的是将它重新塑造成一颗红黑树。
     *
     * 参数说明：
     *     node 插入的结点        // 对应《算法导论》中的z
     */
    private void insertFixUp(RBTNode<E> node) {
        RBTNode<E> parent, gparent;

        // 若“父节点存在，并且父节点的颜色是红色”
        while (((parent = parentOf(node))!=null) && isRed(parent)) {
            gparent = parentOf(parent);

            //若“父节点”是“祖父节点的左孩子”
            if (parent == gparent.left) {
                // Case 1条件：叔叔节点是红色
                RBTNode<E> uncle = gparent.right;
                if ((uncle!=null) && isRed(uncle)) {
                    setBlack(uncle);
                    setBlack(parent);
                    setRed(gparent);
                    node = gparent;
                    continue;
                }

                // Case 3条件：叔叔是黑色，且当前节点是右孩子
                if (parent.right == node) {
                    RBTNode<E> tmp;
                    leftRotate(parent);
                    tmp = parent;
                    parent = node;
                    node = tmp;
                }

                // Case 2条件：叔叔是黑色，且当前节点是左孩子。
                setBlack(parent);
                setRed(gparent);
                rightRotate(gparent);
            } else {    //若“z的父节点”是“z的祖父节点的右孩子”
                // Case 1条件：叔叔节点是红色
                RBTNode<E> uncle = gparent.left;
                if ((uncle!=null) && isRed(uncle)) {
                    setBlack(uncle);
                    setBlack(parent);
                    setRed(gparent);
                    node = gparent;
                    continue;
                }

                // Case 2条件：叔叔是黑色，且当前节点是左孩子
                if (parent.left == node) {
                    RBTNode<E> tmp;
                    rightRotate(parent);
                    tmp = parent;
                    parent = node;
                    node = tmp;
                }

                // Case 3条件：叔叔是黑色，且当前节点是右孩子。
                setBlack(parent);
                setRed(gparent);
                leftRotate(gparent);
            }
        }

        // 将根节点设为黑色
        setBlack(this.root);
    }

    /*
     * 删除结点(z)，并返回被删除的结点
     *
     * 参数说明：
     *     tree 红黑树的根结点
     *     z 删除的结点
     */
    public void remove(E key) {
        RBTNode<E> node;

        if ((node = search(root, key)) != null)
            remove(node);
    }


    /*
     * 删除结点(node)，并返回被删除的结点
     *
     * 参数说明：
     *     node 删除的结点
     */
    private void remove(RBTNode<E> node) {
        RBTNode<E> child, parent;
        boolean color;

        // 被删除节点的"左右孩子都不为空"的情况。
        if ( (node.left!=null) && (node.right!=null) ) {
            // 被删节点的后继节点。(称为"取代节点")
            // 用它来取代"被删节点"的位置，然后再将"被删节点"去掉。
            RBTNode<E> replace = node;

            // 获取后继节点
            replace = replace.right;
            while (replace.left != null)
                replace = replace.left;

            // "node节点"不是根节点(只有根节点不存在父节点)
            if (parentOf(node)!=null) {
                if (parentOf(node).left == node)
                    parentOf(node).left = replace;
                else
                    parentOf(node).right = replace;
            } else {
                // "node节点"是根节点，更新根节点。
                this.root = replace;
            }

            // child是"取代节点"的右孩子，也是需要"调整的节点"。
            // "取代节点"肯定不存在左孩子！因为它是一个后继节点。
            child = replace.right;
            parent = parentOf(replace);
            // 保存"取代节点"的颜色
            color = colorOf(replace);

            // "被删除节点"是"它的后继节点的父节点"
            if (parent == node) {
                parent = replace;
            } else {
                // child不为空
                if (child!=null)
                    setParent(child, parent);
                parent.left = child;

                replace.right = node.right;
                setParent(node.right, replace);
            }

            replace.parent = node.parent;
            replace.color = node.color;
            replace.left = node.left;
            node.left.parent = replace;

            if (color == BLACK)
                removeFixUp(child, parent);

            node = null;
            return ;
        }

        if (node.left !=null) {
            child = node.left;
        } else {
            child = node.right;
        }

        parent = node.parent;
        // 保存"取代节点"的颜色
        color = node.color;

        if (child!=null)
            child.parent = parent;

        // "node节点"不是根节点
        if (parent!=null) {
            if (parent.left == node)
                parent.left = child;
            else
                parent.right = child;
        } else {
            this.root = child;
        }

        if (color == BLACK)
            removeFixUp(child, parent);
        node = null;
    }

    /*
     * 红黑树删除修正函数
     *
     * 在从红黑树中删除插入节点之后(红黑树失去平衡)，再调用该函数；
     * 目的是将它重新塑造成一颗红黑树。
     *
     * 参数说明：
     *     node 待修正的节点
     */
    private void removeFixUp(RBTNode<E> node, RBTNode<E> parent) {
        RBTNode<E> other;

        while ((node==null || isBlack(node)) && (node != this.root)) {
            if (parent.left == node) {
                other = parent.right;
                if (isRed(other)) {
                    // Case 1: x的兄弟w是红色的
                    setBlack(other);
                    setRed(parent);
                    leftRotate(parent);
                    other = parent.right;
                }

                if ((other.left==null || isBlack(other.left)) &&
                        (other.right==null || isBlack(other.right))) {
                    // Case 2: x的兄弟w是黑色，且w的俩个孩子也都是黑色的
                    setRed(other);
                    node = parent;
                    parent = parentOf(node);
                } else {

                    if (other.right==null || isBlack(other.right)) {
                        // Case 4: x的兄弟w是黑色的，并且w的左孩子是红色，右孩子为黑色。
                        setBlack(other.left);
                        setRed(other);
                        rightRotate(other);
                        other = parent.right;
                    }
                    // Case 3: x的兄弟w是黑色的；并且w的右孩子是红色的，左孩子任意颜色。
                    setColor(other, colorOf(parent));
                    setBlack(parent);
                    setBlack(other.right);
                    leftRotate(parent);
                    node = this.root;
                    break;
                }
            } else {

                other = parent.left;
                if (isRed(other)) {
                    // Case 1: x的兄弟w是红色的
                    setBlack(other);
                    setRed(parent);
                    rightRotate(parent);
                    other = parent.left;
                }

                if ((other.left==null || isBlack(other.left)) &&
                        (other.right==null || isBlack(other.right))) {
                    // Case 2: x的兄弟w是黑色，且w的俩个孩子也都是黑色的
                    setRed(other);
                    node = parent;
                    parent = parentOf(node);
                } else {

                    if (other.left==null || isBlack(other.left)) {
                        // Case 4: x的兄弟w是黑色的，并且w的左孩子是红色，右孩子为黑色。
                        setBlack(other.right);
                        setRed(other);
                        leftRotate(other);
                        other = parent.left;
                    }

                    // Case 3: x的兄弟w是黑色的；并且w的右孩子是红色的，左孩子任意颜色。
                    setColor(other, colorOf(parent));
                    setBlack(parent);
                    setBlack(other.left);
                    rightRotate(parent);
                    node = this.root;
                    break;
                }
            }
        }

        if (node!=null)
            setBlack(node);
    }


    /**
     * 查询节点
     * @param key
     * @return
     */
    public RBTNode<E> search(E key) {
        return search(root, key);
    }


    /*
     * (递归实现)查找"红黑树x"中键值为key的节点
     */
    private RBTNode<E> search(RBTNode<E> x, E key) {
        if (x==null)
            return x;

        int cmp = key.compareTo(x.key);
        if (cmp < 0)
            return search(x.left, key);
        else if (cmp > 0)
            return search(x.right, key);
        else
            return x;
    }

    /**
     * 中序遍历
     * @param node
     */
    public void middleTreeIterator(RBTNode<E> node){
        if(node != null){
            middleTreeIterator(node.left);//遍历当前节点左子树
            System.out.println("key:" + node.key);
            middleTreeIterator(node.right);//遍历当前节点右子树
        }
    }



    private RBTNode<E> parentOf(RBTNode<E> node) {
        return node!=null ? node.parent : null;
    }
    private boolean colorOf(RBTNode<E> node) {
        return node!=null ? node.color : BLACK;
    }
    private boolean isRed(RBTNode<E> node) {
        return ((node!=null)&&(node.color==RED)) ? true : false;
    }
    private boolean isBlack(RBTNode<E> node) {
        return !isRed(node);
    }
    private void setBlack(RBTNode<E> node) {
        if (node!=null)
            node.color = BLACK;
    }
    private void setRed(RBTNode<E> node) {
        if (node!=null)
            node.color = RED;
    }
    private void setParent(RBTNode<E> node, RBTNode<E> parent) {
        if (node!=null)
            node.parent = parent;
    }
    private void setColor(RBTNode<E> node, boolean color) {
        if (node!=null)
            node.color = color;
    }


    /*
     * 对红黑树的节点(x)进行左旋转
     *
     * 左旋示意图(对节点x进行左旋)：
     *      px                              px
     *     /                               /
     *    x                               y
     *   /  \      --(左旋)-.           / \                #
     *  lx   y                          x  ry
     *     /   \                       /  \
     *    ly   ry                     lx  ly
     *
     *
     */
    private void leftRotate(RBTNode<E> x) {
        // 设置x的右孩子为y
        RBTNode<E> y = x.right;

        // 将 “y的左孩子” 设为 “x的右孩子”；
        // 如果y的左孩子非空，将 “x” 设为 “y的左孩子的父亲”
        x.right = y.left;
        if (y.left != null)
            y.left.parent = x;

        // 将 “x的父亲” 设为 “y的父亲”
        y.parent = x.parent;

        if (x.parent == null) {
            this.root = y;            // 如果 “x的父亲” 是空节点，则将y设为根节点
        } else {
            if (x.parent.left == x)
                x.parent.left = y;    // 如果 x是它父节点的左孩子，则将y设为“x的父节点的左孩子”
            else
                x.parent.right = y;    // 如果 x是它父节点的左孩子，则将y设为“x的父节点的左孩子”
        }

        // 将 “x” 设为 “y的左孩子”
        y.left = x;
        // 将 “x的父节点” 设为 “y”
        x.parent = y;
    }

    /*
     * 对红黑树的节点(y)进行右旋转
     *
     * 右旋示意图(对节点y进行左旋)：
     *            py                               py
     *           /                                /
     *          y                                x
     *         /  \      --(右旋)-.            /  \                     #
     *        x   ry                           lx   y
     *       / \                                   / \                   #
     *      lx  rx                                rx  ry
     *
     */
    private void rightRotate(RBTNode<E> y) {
        // 设置x是当前节点的左孩子。
        RBTNode<E> x = y.left;

        // 将 “x的右孩子” 设为 “y的左孩子”；
        // 如果"x的右孩子"不为空的话，将 “y” 设为 “x的右孩子的父亲”
        y.left = x.right;
        if (x.right != null)
            x.right.parent = y;

        // 将 “y的父亲” 设为 “x的父亲”
        x.parent = y.parent;

        if (y.parent == null) {
            this.root = x;            // 如果 “y的父亲” 是空节点，则将x设为根节点
        } else {
            if (y == y.parent.right)
                y.parent.right = x;    // 如果 y是它父节点的右孩子，则将x设为“y的父节点的右孩子”
            else
                y.parent.left = x;    // (y是它父节点的左孩子) 将x设为“x的父节点的左孩子”
        }

        // 将 “y” 设为 “x的右孩子”
        x.right = y;

        // 将 “y的父节点” 设为 “x”
        y.parent = x;
    }
}
```

### B树（B-tree）
###### 来源
B树又叫平衡多路查找树，不是二叉的，和**B+树一样，其中的B代表平衡（balance），不是binary**  
现在肯定知道，树的查询时间复杂度和其树的高度有直接关系，当向红黑树的数据结构里面插入大量的数据时，有两个问题：  

（1）首先，内存是有限的不可能无止境的一直插入数据，而基于BST的平衡树如AVL树和红黑树本质上都是基于内存的数据结构，内存可能会吃不消  

（2）如果将AVL树或者红黑树或者跳跃表直接存储到磁盘上，依旧是可以用来检索的。但问题是基于磁盘寻道查询比基于内存慢太多了，举个例子，假设现在有100万数据分布在红黑树里面，按照红黑树的平均查找性能O（logN）来计算，在N=100万时，检索任意数据平均需要20次查询，如果是在内存完全没问题。但此时红黑树是如果存在磁盘上，那就完全不一样了，按照普通硬盘或者文件系统，每次查询IO一次总耗时约10毫秒计算，那么在百万级别的数据量下，检索一次数据就大概就是0.2秒，假如现在数据量扩大100倍，数据总量是一亿，那么检索一次数据就需要20秒，这个性能是完全不能接受的，可以想象在这种情况下，使用二叉树是不合适的  

所以迫切需要一种面向磁盘或者文件系统友好的数据结构，而它就是B树，或者基于B树的扩展B+，B\*树等  

这个时候可以分析一下了，上面问题的根源在于树的高度太高，导致查询外存，也就是访问磁盘IO次数太多，从而大大拖慢了检索性能，那么思路就清晰了，**只要想法子降低树的高度就可以了**  
没错，B树就是这么一种多叉平衡树，你可以想象它的孩子节点少则2个，多则数千个，所以就从二叉树的廋高，变成了多叉树的矮胖，正是这样才大大降低了树的高度，从而使得这种数据结构更适合存储在磁盘上，并且充分了利用了磁盘的块的预读机制（局部性访问原理），通过缓存的方式进一步提升性能，磁盘在读取某一个文件指针的时候，通常会把紧挨着的数据，也全部读到缓存，因为实践证明，当一个文件数据被访问的时候，它周边的数据很快也会被访问  

##### 磁盘存储结构
- 扇区：磁盘的最小存储单位  
- 磁盘块（block）
文件系统读写数据的最小单位，系统从磁盘读取数据到内存时是以磁盘块为基本单位的，位于同一个磁盘块中的数据会被一次性读取出来，而不是需要什么取什么   

InnoDB存储引擎中有页（Page）的概念，页是其磁盘管理的最小单位。InnoDB存储引擎中默认每个页的大小为16KB，可通过参数innodb_page_size将页的大小设置为4K、8K、16K，在MySQL中可通过如下命令查看页的大小：  
```sql
mysql> show variables like 'innodb_page_size';
```
系统一个磁盘块的存储空间往往没有这么大，因此InnoDB每次申请磁盘空间时都会是若干地址连续磁盘块来达到页的大小16KB。InnoDB在把磁盘数据读入到内存时会以页为基本单位，在查询数据时如果一个页中的每条数据都能有助于定位数据记录的位置，这将会减少磁盘I/O次数，提高查询效率，而B-Tree结构的数据可以让系统高效的找到数据所在的磁盘块  

##### 定义
为了描述B-Tree，首先定义一条记录为一个二元组\[key, data\] ，key为记录的键值，对应表中的主键值，data为一行记录中除主键外的数据。对于不同的记录，key值互不相同  
假如当前有一颗m阶的B树（注意阶的意思是指节点的孩子节点个数上限），那么其符合：  
- 每个节点最多有m个子节点  
- 除了根节点和叶子节点之外，其他的每个节点最少有m/2（向上取整）个孩子节点  
- 根节点至少有两个孩子节点，（除了第一次插入的时候，此时只有一个节点，根节点同时是叶子节点）  
- 所有的叶子节点都在同一层，且不包含其它关键字信息   
- 有k个子节点的父节点包含k-1个关键字  

##### 结构优势
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/bt-image-wiki.jpg)

B-Tree中的每个节点根据实际情况可以包含大量的关键字信息和分支，如下图所示为一个3阶的B-Tree：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/bt-image-01.jpg)

每个节点占用一个盘块的磁盘空间，一个节点上有**两个升序排序的关键字**和**三个指向子树根节点的指针**，指针存储的是子节点所在磁盘块的地址。两个关键词划分成的三个范围域对应三个指针指向的子树的数据的范围域。以根节点为例，关键字为17和35，P1指针指向的子树的数据范围为小于17，P2指针指向的子树的数据范围为17~35，P3指针指向的子树的数据范围为大于35

模拟查找关键字29的过程：  
1. 根据根节点找到磁盘块1，读入内存。【磁盘I/O操作第1次】  
2. 比较关键字29在区间（17,35），找到磁盘块1的指针P2  
3. 根据P2指针找到磁盘块3，读入内存。【磁盘I/O操作第2次】  
4. 比较关键字29在区间（26,30），找到磁盘块3的指针P2  
5. 根据P2指针找到磁盘块8，读入内存。【磁盘I/O操作第3次】  
6. 在磁盘块8中的关键字列表中找到关键字29  

分析上面过程，发现需要3次磁盘I/O操作，和3次内存查找操作。由于内存中的关键字是一个有序表结构，可以利用二分法查找提高效率。而3次磁盘I/O操作是影响整个B-Tree查找效率的决定因素。B-Tree相对于AVLTree缩减了节点个数，使每次磁盘I/O取到内存的数据都发挥了作用，从而提高了查询效率  

##### 其他特性
- 树高平衡，所有的叶节点都在同一层  
- 关键码没有重复，父节点中的关键码是其子节点的分解  
- B树把值接近的相关记录放在同一个磁盘页中，从而利用了访问的局部性原理  
- B树保证树种至少有一部分比例的节点是满的  
为什么这样说，在上面的性质2中，知道每个节点最少可以有m/2个节点，注意这刚好是一半，没有太满，是因为可以给后续的添加，删除留有余地，这样以来节点不会频繁的触发不平衡，没有太空则意味着B树能够保证降低树的高度  

### B+树（B+ tree）
B树的不足也显而易见，在B树里面所有的节点都存储数据，这样导致在做范围查询的时候，性能比较低

此外，从B-Tree结构图中可以看到每个节点中不仅包含数据的key值，还有data值。而每一个页的存储空间是有限的，如果data数据较大时将会导致每个节点（即一个页）能存储的key的数量很小，当存储的数据量很大时同样会导致B-Tree的深度较大，增大查询时的磁盘I/O次数，进而影响查询效率

所以才出现了B+树，例如InnoDB存储引擎就是用B+Tree实现其索引结构。具体的原理是，在B+Tree中，所有数据记录节点都是按照键值大小顺序存放在同一层的叶子节点上，而非叶子节点上只存储key值信息，这样可以大大加大每个节点存储的key值数量，降低B+Tree的高度  
##### 定义
相对于B-Tree而言：  
1. 非叶子节点只存储键值信息  
2. 所有叶子节点之间都有一个链指针  
3. 所有数据记录都存放在叶子节点中  

##### 结构优势
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/b+tree-image-wiki.jpg)

将前面的B-Tree变为B+Tree后如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/b+tree-image-01.jpg)

通常在B+Tree上有两个头指针，一个指向根节点，另一个指向关键字最小的叶子节点，而且所有叶子节点（即数据节点）之间是一种链式环结构

因此可以对B+Tree进行两种查找运算：一种是对于主键的范围查找和分页查找，另一种是从根节点开始，进行随机查找

举个例子来证明B+Tree的优势：  
InnoDB存储引擎中页的大小为16KB，一般表的主键类型为INT（占用4个字节）或BIGINT（占用8个字节），指针类型也一般为4或8个字节，也就是说一个页（B+Tree中的一个节点）中大概存储16KB/(8B+8B)=1K个键值（因为是估值，为方便计算，这里的K取值为〖10〗^3）。也就是说一个深度为3的B+Tree索引可以维护10^3 \* 10^3 \* 10^3 = 10亿 条记录  

实际情况中每个节点可能不能填充满，因此在数据库中，B+Tree的高度一般都在2~4层。mysql的InnoDB存储引擎在设计时是将根节点常驻内存的，也就是说查找某一键值的行记录时最多只需要1~3次磁盘I/O操作  

数据库中的B+Tree索引可以分为聚集索引（clustered index）和辅助索引（secondary index）。上面的B+Tree示例图在数据库中的实现即为聚集索引，聚集索引的B+Tree中的叶子节点存放的是整张表的行记录数据。辅助索引与聚集索引的区别在于辅助索引的叶子节点并不包含行记录的全部数据，而是存储相应行数据的聚集索引键，即主键。当通过辅助索引来查询数据时，InnoDB存储引擎会遍历辅助索引找到主键，然后再通过主键在聚集索引中找到完整的行记录数据  

### B*树（B* tree）
是B+树的变体，在B+树的非根和非叶子结点再增加指向兄弟的指针  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/b-star-image-01.jpg)

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/b-star-image-02.jpg)

B\*树定义了非叶子结点关键字个数至少为(2/3)\*M，即块的最低使用率为2/3。对比之下，B+树为1/2  

##### B+树的分裂  
当一个结点满时，分配一个新的结点，并将原结点中1/2的数据复制到新结点，最后在父结点中增加新结点的指针；B+树的分裂只影响原结点和父结点，而不会影响兄弟结点，所以它不需要指向兄弟的指针  
##### B*树的分裂  
当一个结点满时，如果它的下一个兄弟结点未满，那么将一部分数据移到兄弟结点中，再在原结点插入关键字，最后修改父结点中兄弟结点的关键字（因为兄弟结点的关键字范围改变了）；如果兄弟也满了，则在原结点与兄弟结点之间增加新结点，并各复制1/3的数据到新结点，最后在父结点增加新结点的指针  

**所以，B\*树分配新结点的概率比B+树要低，空间使用率更高**




 







