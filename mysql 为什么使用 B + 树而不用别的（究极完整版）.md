# mysql 为什么使用 B + 树而不用别的（究极完整版）
# MySQL为什么使用树结构？

文件很大，不可能全部存储在内存中，故要存储到磁盘上  
索引的结构组织要尽量减少查找过程中磁盘I/O的存取次数（为什么使用B-/+Tree，还跟磁盘存取原理有关）  
局部性原理与磁盘预读，预读的长度一般为页（page）的整倍数（操作系统内存页的大小通常为4k）。其中MySQL B+树中的 叶/非叶节点 都是以MySQL的页为单位（大小通常也为16k），存放完整行记录。  
数据库系统巧妙利用了磁盘预读原理，将一个节点大小设为操作系统内存页的整数倍，这样每个节点只需要一次I/O就可以完全载入。而红黑树这种结构，高度明显要深的多。由于逻辑上很近的节点（父子）物理上可能很远，无法利用局部性。

> 这里我们就需要了解页（page）的概念，在计算机里，无论是内存还是磁盘，操作系统都是按页的大小进行读取的（页大小通常为 4 kb），
> 
> 磁盘每次读取都会预读，会提前将连续的数据读入内存中，这样就避免了多次 IO，这就是计算机中有名的局部性原理，
> 
> 即我用到一块数据，很大可能这块数据附近的数据也会被用到，干脆一起加载，省得多次 IO 拖慢速度  
> 这个连续数据有多大呢，必须是操作系统页大小的整数倍，
> 
> 这个连续数据就是 MySQL 的页，默认值为 16 KB，也就是说对于 B+ 树的节点，
> 
> 最好设置成页的大小（16 KB），这样一个 B+ 树上的节点就只会有一次 IO 读。
> 
> 那有人就会问了，这个页大小是不是越大越好呢，设置大一点，节点可容纳的数据就越多，树高越小，IO 不就越小了吗  
> 这里要注意，页大小并不是越大越好，InnoDB 是通过内存中的缓存池（pool buffer）来管理从磁盘中读取的页数据的。
> 
> 页太大的话，很快就把这个缓存池撑满了，可能会造成页在内存与磁盘间频繁换入换出，影响性能。  
> 通过以上分析，相信我们不难猜测出 N 叉树中的 N 该怎么设置了，
> 
> 只要选的时候尽量保证每个节点的大小等于一个页（16kb）的大小即可。

> 参考：磁盘读取
> 
> 计算机系统是分页读取和存储的，一般一页为4KB（8个扇区，每个扇区512B，8*512B=4KB），每次读取和存取的最小单元为一页，而磁盘预读时通常会读取页的整倍数。
> 
> 根据文章上述的【局部性原理】①当一个数据被用到时，其附近的数据也通常会马上被使用。②程序运行期间所需要的数据通常比较集中。由于磁盘顺序读取的效率很高（不需要寻道时间，只需很少的旋转时间），所以即使只需要读取一个字节，磁盘也会读取一页的数据。
> 
> MySQL InnoDB默认的页大小为**16k**（可通常 innodb\_page\_size 参数设置），而操作系统中的磁盘页大小通常为4k，所以这里可以认为MySQL InnoDB中的1页（1个磁盘块）相当于操作系统中的4页。
> 
> 这也就符合MySQL InnoDB所利用到的磁盘预读通常会预读操作系统页的整数倍（4倍）。具体如下图：
> 
> ![](https://oscimg.oschina.net/oscnet/up-74c2b3ab84a95a9798afb037ff4461786a3.png)  
>        最外层浅蓝色磁盘块1里有数据17、35（深蓝色）和指针P1、P2、P3（黄色）。P1指针表示小于17的磁盘块，P2是在17-35之间，P3指向大于35的磁盘块。真实数据存在于叶子节点也就是最底下的一层3、5、9、10、13......非叶子节点不存储真实的数据，只存储指引搜索方向的数据项，如17、35。
> 
> 查找过程：例如搜索28数据项，首先加载磁盘块1到内存中，发生一次I/O，用二分查找确定在P2指针。接着发现28在26和30之间，通过P2指针的地址加载磁盘块3到内存，发生第二次I/O。用同样的方式找到磁盘块8，发生第三次I/O。
> 
> 真实的情况是，上面3层的B+Tree可以表示2k万的数据，这种量级的数据只发生了三次I/O，时间提升是巨大的。

## 什么是B树

#### 一个m阶的B树具有如下几个特征：

> 1、根结点至少有两个子女。  
> 2、每个中间节点都包含k-1个元素和k个孩子，其中 m/2 <= k <= m  
> 3、每一个叶子节点都包含k-1个元素，其中 m/2 <= k <= m  
> 4、所有的叶子结点都位于同一层。  
> 5、每个节点中的元素从小到大排列，节点当中k-1个元素正好是k个孩子包含的元素的值域分划。

## 什么是B+树

一个m阶的B+树具有如下几个特征：

> 1、有k个子树的中间节点包含有k个元素（B树中是k-1个元素），每个元素不保存数据，只用来索引，所有数据都保存在叶子节点。  
> 2、所有的叶子结点中包含了全部元素的信息，及指向含这些元素记录的指针，且叶子结点本身依关键字的大小自小而大顺序链接。  
> 3、所有的中间节点元素都同时存在于子节点，在子节点元素中是最大（或最小）元素。  
> 4\. 每个节点中子节点的个数不能超过 N，也不能小于 N/2（不然会造成页分裂或页合并）  
> 5\. 根节点的子节点个数可以不超过 m/2，这是一个例外

# B树和B+树的区别：

1.  B树的每个节点都存储了key和data，而B+树的data存储在叶子节点上。  
    B+树非叶子节点仅存储key不存储data，这样一个节点就可以存储更多的key。可以使得B+树相对B树来说更矮（IO次数就是树的高度），所以与磁盘交换的IO操作次数更少。
2.  B+树所有叶子节点构成一个有序链表，按主键排序来遍历全部记录，能更好支持范围查找。  
    由于数据顺序排列并且相连，所以便于区间查找和搜索。而B树则需要进行每一层的递归遍历，相邻的元素可能在内存中不相邻，所以缓存命中性没有B+树好。
3.  B+树所有的查询都要从根节点查找到叶子节点，查询性能更稳定；而B树，每个节点都可能查找到数据，需要在叶子节点和内部节点不停的往返移动，所以不稳定。

## 三个维度优化索引

下面我们从以下三个维度优化下索引的设计

**(1)首先我们从时间角度上**

我们需要为了避免频繁的页分裂，需要尽可能使用主键自增长等方式，保证新增的数据页中的数据行的主键都是递增，避免不必要的页分裂带来的性能损耗和拖慢查询效率。

> 为啥推荐自增 id 作为主键，自建主键不行吗，
> 
> 有人可能会说用户的身份证是唯一的，可以用它来做主键，假设以身份证作主键，会有什么问题呢。  
> B+ 树为了维护索引的有序性，每插入或更新一条记录的时候，会对索引进行更新。
> 
> 假设原来基于身份证作索引的 B+ 树如下（假设为二叉树 ，图中只列出了身份证的前四位）
> 
> ![](https://img2020.cnblogs.com/blog/72194/202103/72194-20210312231351756-1545542800.png)
> 
> 现在有一个开头是 3604 的身份证对应的记录插入 db ，
> 
> 此时要更新索引，按排序来更新的话，
> 
> 显然这个 3604 的身份证号应该插到左边节点 3504 后面（如下图示，假设为二叉树）
> 
> ![](https://img2020.cnblogs.com/blog/72194/202103/72194-20210312231434979-2000730129.png)
> 
> 如果把 3604 这个身份证号插入到 3504 后面的话，这个节点的元素个数就有 3 个了，
> 
> 显然不符合二叉树的条件，此时就会造成页分裂，
> 
> 就需要调整这个节点以让它符合二叉树的条件
> 
> ![](https://img2020.cnblogs.com/blog/72194/202103/72194-20210312231616392-1015391359.png)
> 
> 这种由于页分裂造成的调整必然导致性能的下降，尤其是以身份证作为主键的话，由于身份证的随机性，
> 
> 必然造成大量的随机结点中的插入，进而造成大量的页分裂，进而造成性能的急剧下降  
> 那如果是以自增 id 作为主键呢，由于新插入的表中生成的 id 比索引中所有的值都大，
> 
> 所以它要么合到已存在的节点（元素个数未满）中，
> 
> 要么放入新建的节点中（如下图示）所以如果是以自增 id 作为主键，就不存在页分裂的问题了，推荐！
> 
> ![](https://img2020.cnblogs.com/blog/72194/202103/72194-20210312231804304-2026735356.png)
> 
> 有页分裂就必然有页合并，什么时候会发生页合并呢，
> 
> 当删除表记录的时候，索引也要删除，此时就有可能发生页合并，如图示
> 
> ![](https://img2020.cnblogs.com/blog/72194/202103/72194-20210312231904803-1697960883.png)
> 
> 当我们删除 id 为 7，9 对应行的时候，上图中的索引就要更新，
> 
> 把 7，9 删掉，此时 8，10 就应该合到一个节点，不然 8，10 分散在两个节点上，
> 
> 可能造成两次 IO 读，势必会影响查找效率!  
> 那什么时候会发生页合并呢，我们可以定个阈值，
> 
> 比如对于 N 叉树来说，
> 
> 当节点的个数小于 N/2 的时候就应该和附近的节点合并，
> 
> 不过需要注意的是合并后节点里的元素大小可能会超过 N，造成页分裂，
> 
> 需要再对父节点等进行调整以让它满足 N 叉树的条件。

另外选择合适的字段作为索引字段也很重要，需要选择基数较大的字段，也就是一个字段可能出现的值比较多，这样我们在B+树中查询时，才能最高效的发挥出二分法查询的威力，如果建立索引的字段基数比较小可能查询时二分查找就会退化成时间复杂度为O(n)的线性查询了。

**(2)空间的角度上**

因为索引数据本身也是要占空间的，可以选择字段长度较小的作为索引字段，这样整棵B+树不至于那么占空间。

但是如果非得要以长字段作为索引也不是不行，可以采用折中的以字段的前缀作为索引，这样的索引也称为前缀索引，但是这样可能只能用在模糊查询上了，用在group by和order by上就不太适合了。

**(3)作用范围上**

当然我们设计索引的目的，当然是为了更好的用上索引，索引在设计时，尽可能让where、group by、order by这些语句都能用上索引。

### Mysql的索引为什么使用B+树而不使用跳表?

B+树是多叉树结构，每个结点都是一个16k的数据页，能存放较多索引信息，所以扇出很高。三层左右就可以存储2kw左右的数据(知道结论就行，想知道原因可以看之前的文章)。也就是说查询一次数据，如果这些数据页都在磁盘里，那么最多需要查询三次磁盘IO。

跳表是链表结构，一条数据一个结点，如果最底层要存放2kw数据，且每次查询都要能达到二分查找的效果，2kw大概在2的24次方左右，所以，跳表大概高度在24层左右。最坏情况下，这24层数据会分散在不同的数据页里，也即是查一次数据会经历24次磁盘IO。

因此存放同样量级的数据，B+树的高度比跳表的要少，如果放在mysql数据库上来说，就是磁盘IO次数更少，因此B+树查询更快。

而针对写操作，B+树需要拆分合并索引数据页，跳表则独立插入，并根据随机函数确定层数，没有旋转和维持平衡的开销，因此跳表的写入性能会比B+树要好。

其实，mysql的存储引擎是可以换的，以前是myisam，后来才有的innodb，它们底层索引用的都是B+树。也就是说，你完全可以造一个索引为跳表的存储引擎装到mysql里。事实上，facebook造了个rocksDB的存储引擎，里面就用了跳表。直接说结论，它的写入性能确实是比innodb要好，但读性能确实比innodb要差不少。感兴趣的话，可以在文章最后面的参考资料里看到他们的性能对比数据。

### redis为什么使用跳表而不使用B+树或二叉树呢?

redis支持多种数据结构，里面有个有序集合，也叫ZSET。内部实现就是跳表。那为什么要用跳表而不用B+树等结构呢?

这个几乎每次面试都要被问一下。

虽然已经很熟了，但每次都要装作之前没想过，现场思考一下才知道答案。

真的，很考验演技。

大家知道，redis 是纯纯的内存数据库。

进行读写数据都是操作内存，跟磁盘没啥关系，因此也不存在磁盘IO了，所以层高就不再是跳表的劣势了。

并且前面也提到B+树是有一系列合并拆分操作的，换成红黑树或者其他AVL树的话也是各种旋转，目的也是为了保持树的平衡。

而跳表插入数据时，只需要随机一下，就知道自己要不要往上加索引，根本不用考虑前后结点的感受，也就少了旋转平衡的开销。

因此，redis选了跳表，而不是B+树。

## 面试题

**问题1：MySQL中存储索引用到的数据结构是B+树，B+树的查询时间跟树的高度有关，是log(n)，如果用hash存储，那么查询时间是O(1)。既然hash比B+树更快，为什么mysql用B+树来存储索引呢？**

**答：**一、从内存角度上说，数据库中的索引一般时在磁盘上，数据量大的情况可能无法一次性装入内存，B+树的设计可以允许数据分批加载。

二、从业务场景上说，如果只选择一个数据那确实是hash更快，但是数据库中经常会选中多条这时候由于B+树索引有序，并且又有链表相连，它的查询效率比hash就快很多了。

**问题2：为什么不用红黑树或者二叉排序树？**

**答：**树的查询时间跟树的高度有关，B+树是一棵多路搜索树可以降低树的高度，提高查找效率

**问题3：**B+树每一层越宽越好吗

B+树每一层越宽并不一定越好。虽然增加每个节点的大小可以减少磁盘I/O的次数，但是如果节点太大，可能会导致内存无法一次性加载一个节点，从而导致额外的磁盘I/O操作，降低性能。此外，节点太大还会导致磁盘碎片问题，降低磁盘的空间利用率。因此，在选择节点大小时，需要权衡节点大小和磁盘I/O次数之间的关系，并根据具体的应用场景进行调整

**问题4：在内存中，红黑树比B树更优，但是涉及到磁盘操作B树就更优了，那么你能讲讲B+树吗？**

B+树是在B树的基础上进行改造，它的数据都在叶子结点，同时叶子结点之间还加了指针形成链表。

下面是一个4路B+树，它的数据都在叶子结点，并且有链表相连。

![](https://oscimg.oschina.net/oscnet/up-65a351b27c32b5487c1ec0ad6c11370d66e.png)![](https://segmentfault.com/img/remote/1460000021488915)

**问题5：为什么B+树要这样设计？**

**答：**这个跟它的使用场景有关，B+树在数据库的索引中用得比较多，数据库中select数据，不一定只选一条，很多时候会选中多条，比如按照id进行排序后选100条。如果是多条的话，B树需要做局部的中序遍历，可能要跨层访问。而B+树由于所有数据都在叶子结点不用跨层，同时由于有链表结构，只需要找到首尾，通过链表就能把所有数据取出来了。

比如选出7到19只需要在叶子结点中就能找到。

![](https://oscimg.oschina.net/oscnet/up-ea24fbe229049e9d73aacf1743f71bb0c56.png)![](https://segmentfault.com/img/remote/1460000021488914)

## 总结

MySQL 是会将数据持久化在硬盘，而存储功能是由 MySQL 存储引擎实现的，所以讨论 MySQL 使用哪种数据结构作为索引，实际上是在讨论存储引使用哪种数据结构作为索引，InnoDB 是 MySQL 默认的存储引擎，它就是采用了 B+ 树作为索引的数据结构。

要设计一个 MySQL 的索引数据结构，不仅仅考虑数据结构增删改的时间复杂度，更重要的是要考虑磁盘 I/0 的操作次数。因为索引和记录都是存放在硬盘，硬盘是一个非常慢的存储设备，我们在查询数据的时候，最好能在尽可能少的磁盘 I/0 的操作次数内完成。

二分查找树虽然是一个天然的二分结构，能很好的利用二分查找快速定位数据，但是它存在一种极端的情况，每当插入的元素都是树内最大的元素，就会导致二分查找树退化成一个链表，此时查询复杂度就会从 O(logn)降低为 O(n)。

为了解决二分查找树退化成链表的问题，就出现了自平衡二叉树，保证了查询操作的时间复杂度就会一直维持在 O(logn) 。但是它本质上还是一个二叉树，每个节点只能有 2 个子节点，随着元素的增多，树的高度会越来越高。

而树的高度决定于磁盘 I/O 操作的次数，因为树是存储在磁盘中的，访问每个节点，都对应一次磁盘 I/O 操作，也就是说树的高度就等于每次查询数据时磁盘 IO 操作的次数，所以树的高度越高，就会影响查询性能。

B 树和 B+ 都是通过多叉树的方式，会将树的高度变矮，所以这两个数据结构非常适合检索存于磁盘中的数据。

但是 MySQL 默认的存储引擎 InnoDB 采用的是 B+ 作为索引的数据结构，原因有：

-   B+ 树的非叶子节点不存放实际的记录数据，仅存放索引，因此数据量相同的情况下，相比存储即存索引又存记录的 B 树，B+树的非叶子节点可以存放更多的索引，因此 B+ 树可以比 B 树更「矮胖」，查询底层节点的磁盘 I/O次数会更少。
-   B+ 树有大量的冗余节点（所有非叶子节点都是冗余索引），这些冗余索引让 B+ 树在插入、删除的效率都更高，比如删除根节点的时候，不会像 B 树那样会发生复杂的树的变化；
-   B+ 树叶子节点之间用链表连接了起来，有利于范围查询，而 B 树要实现范围查询，因此只能通过树的遍历来完成范围查询，这会涉及多个节点的磁盘 I/O 操作，范围查询效率不如 B+ 树。

[MySQL B+树相对于B树的区别及优势](https://blog.csdn.net/qq_37102984/article/details/119646611)

[为什么mysql索引要使用B+树，而不是B树，红黑树](https://segmentfault.com/a/1190000021488885)

[为什么 MySQL 采用 B+ 树作为索引？](https://xiaolincoding.com/mysql/index/why_index_chose_bpuls_tree.html#%E6%80%8E%E6%A0%B7%E7%9A%84%E7%B4%A2%E5%BC%95%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E6%98%AF%E5%A5%BD%E7%9A%84)

[Mysql 索引模型 B+ 树详解](https://segmentfault.com/a/1190000022192940)