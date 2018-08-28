## 可随机读写的队列模型

背景:在我们的业务场景中,经常会遇见队列这种数据结构,他是一个满足先进先出原则的容器。队列可以用来实现任务缓存,数据轮询。
而我们抓取业务经常需要使用一些任务资源,为了让这些资源均匀使用,我们使用队列,使得资源可以被one by one的分发。但是在遇到
抓取资源存在不好的反馈是,我们希望可以调整队列顺序,用来实现资源的权值调整,达到坏的资源低频次使用的效果。所以,我对现存队列
模型做了一个限制性约束,使得该模型能够更加方便的支持可以调整轮询资源权值的需求。即需要一个数据容器模型,1)满足数据可以被轮询,
也即在某一个时刻,得到某一个元素,那么再一次得到该元素应该是其他所有元素全部被访问过之后。2)如果有需要,可以调整容器任意index
的元素的顺序,从而在局部打破第一条约束。


## 模型数据结构选取
我们可以发现,需求1就是常见的队列。所以我们可以使用任何一种队列模型完成需求1。已知两种队列的数据结构包括链表和环形数组。

### 链表
链表是非常容易想到的队列底层数据结构支撑,他就是一个使用指针串联的容器。可以非常方便的在首位增加或者删除元素,也可以在链表中间
插入或者删除元素。所以,如果访问首部元素,将其放置到尾部,则实现了轮询。通过首部指针遍历链表,定位到中部节点,将其自链表摘下,然后寻找
另外的位置插入,则实现了元素顺序修改。但是我们很容易发现,如果需要修改元素顺序,需要遍历链表才能实现,这个在容器元素数量过大的时候,
将会严重影响性能。他的时间复杂度是O(n)。我不知道redis的链表是否是使用链表实现,但是表现来看10w左右资源的时候,将会引起redis压力。
另外当数据量大到无法再内存中缓存的时候,在文件中维护链表,数据结构也是不太好操控的。

### 环形数组
另一种队列的数据结构是环形数组,他就是一片连续的存储空间,使用取模的方式将index约束在空间之内。我们需要记录数组start和end的两个游标。
使用游标移动的方式表示数据的首尾。那么在我们这个场景下,如果容器空间恰好等于元素大小(这个很容易做到,我们可以直接裁剪数组)。那么可以将
start和end合并,使用单个游标即可实现需求1,但是需求2的实现消耗是巨大的,我们虽然可以快速定位到元素,但是由于他不是链表的方式可以随意摘下节点,
我们取出节点之后,搬移需要移动后面的所有元素,看起来也是O(n)的时间复杂度,但是考虑到元素内容可能不是普通数据类型,其内存复制消耗也是挺大的。
环形数组适合用来做消息队列,同时也方便在文件中存储,因为数据空间连续,可以直接将数据区域写入文件。但是明显无法做到高效的在队列中间插入或者删除数据。

### 如何实现在队列中间插入数据,而不太影响原来的数据的存储结构呢?

