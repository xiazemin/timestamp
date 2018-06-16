# LevelDB中LSM Tree的实现细节

由于LevelDB是开源的, 所以从中可以了解到更多的SSTable和LSM tree的实现细节 

LevelDb作为存储系统，其中核心就是SSTable, 下面先看看SSTable在LevelDb中的结构是怎样的...



image





内存 

Memtable, Immutable Memtable 

写入首先写入Memtable, 当Memtable插入的数据占用内存到了一个界限后，需要将内存的记录导出到外存文件中. 

生成新的Log文件和Memtable，原先的Memtable就成为Immutable Memtable，顾名思义，就是说这个Memtable的内容是不可更改的，只能读不能写入或者删除。新到来的数据被记入新的Log文件和Memtable，LevelDb后台调度会将Immutable Memtable的数据导出到磁盘，形成一个新的SSTable文件.



 



磁盘



主要是多level的SSTable\(levelDB也由此得名\), 每个SSTable文件, 以.sst为后缀, 并且文件内的key:value都是按key排序的, 局部有序



Level 0 SSTable, 由Immutable Memtable进行minor compaction得到. 所以level 0比较特殊, SSTable files之间的key range会有重合, 因为是从Memtable compaction生成, 所以无法保证不重合



其他level SSTable, 由上级的SSTable进行major compaction得到, 比如level 1是由level 0 compaction得到



不断把多个低级别的SSTable, compaction到一个高级别的SSTable, 目的是提高读效率, 因为如果需要打开很多的SSTable进行查询, 明显效率会很低. 而经过多level的compaction, 来删除掉一些不再有效的KV数据, 减小数据规模, 减少文件数量等, 使效率大大提高. 



Bigtable中讲到三种类型的compaction: minor, major和full。所谓minor Compaction，就是把memtable中的数据导出到SSTable文件中；major compaction就是合并不同层级的SSTable文件，而full compaction就是将所有SSTable进行合并。LevelDb包含其中两种，minor和major。



 



除了SSTable文件外, 还有3种files,



log文件, 防止数据丢失的



当应用写入一条Key:Value记录的时候，LevelDb会先往log文件里写入，成功后将记录插进Memtable中，这样基本就算完成了写入操作，因为一次写入操作只涉及一次磁盘顺序写和一次内存写入，所以这是为何说LevelDb写入速度极快的主要原因。



Manifest文件，记录各个SSTable文件的元数据, 哪一层,范围



image



Current文件, 记录当前的manifest文件名



因为在LevleDb的运行过程中，随着Compaction的进行，SSTable文件会发生变化，会有新的文件产生，老的文件被废弃，Manifest也会跟着反映这种变化，此时往往会新生成Manifest文件来记载这种变化，而Current则用来指出哪个Manifest文件才是我们关心的那个Manifest文件。



 



log文件结构

LevelDb对于一个log文件，会把它切割成以32K为单位的物理Block，每次读取的单位以一个Block作为基本读取单位. 为什么要分block? 应该出于磁盘读取效率的考虑



记录如果在一个block里面就可以放下, 那么Type就是full, 如A, C



记录如果需要多个block才可以放下, 那么Type分别是, First, Middle, Last 如B



image



至于每条record的逻辑结构如下,



image



 



SSTable文件结构

SSTable, .sst文件的结构, 也是分成固定大小的block. 

除了大部分Data blocks外, 在文件的末端, 还会有一些用于数据管理的blocks…



image



其中比较重要的是, block index, 这个用于有效的提高读效率, 尤其当SSTable比较大的时候



索引的结构如下图, 也很简单



image



其他细节,参考原文



 



MemTable详解

LevelDb的MemTable提供了将KV数据写入，删除以及读取KV记录的操作接口，但是事实上Memtable并不存在真正的删除操作,删除某个Key的Value在Memtable内是作为插入一条记录实施的，但是会打上一个Key的删除标记，真正的删除操作是Lazy的，会在以后的Compaction过程中去掉这个KV。



需要注意的是，LevelDb的Memtable中KV对是根据Key大小有序存储的，在系统插入新的KV时，LevelDb要把这个KV插到合适的位置上以保持这种Key有序性。其实，LevelDb的Memtable类只是一个接口类，真正的操作是通过背后的SkipList来做的，包括插入操作和读取操作等，所以Memtable的核心数据结构是一个SkipList。



SkipList是由William Pugh发明。他在Communications of the ACM June 1990, 33\(6\) 668-676 发表了Skip lists: a probabilistic alternative to balanced trees，在该论文中详细解释了SkipList的数据结构和插入删除操作。SkipList是平衡树的一种替代数据结构，但是和红黑树不相同的是，SkipList对于树的平衡的实现是基于一种随机化的算法的，这样也就是说SkipList的插入和删除的工作是比较高效的.



关于SkipList的详细介绍可以参考这篇文章：http://www.cnblogs.com/xuqiang/archive/2011/05/22/2053516.html，LevelDb的SkipList基本上是一个具体实现，并无特殊之处



SkipList不仅是维护有序数据的一个简单实现，而且相比较平衡树来说，在插入数据的时候可以避免频繁的树节点调整操作，所以写入效率是很高的，LevelDb整体而言是个高写入系统，SkipList在其中应该也起到了很重要的作用。Redis为了加快插入操作，也使用了SkipList来作为内部实现数据结构。



 



SSTable读写操作



写入操作



对于SSTable而言, 插入, 更新, 删除, 都是通过append来实现的, 只不过delete插入的“Key:删除标记”, 后台Compaction的时候才去做真正的删除操作, 如key3



image



 



读操作



SSTable的读操作比较复杂一些, 不过下图还是比较好的反映出读取的过程,



MemTable –&gt; Immutable MemTable –&gt; Level0 SSTable –&gt; Level1 SSTable –&gt; Leveln



这个顺序是有很道理的, 由于SSTable所有的写都是append, 所以同一个key的value可能有很多版本, 而我们只关心最新的那个 

所以我们只要安装这个顺序区读, 就能保证读到最新的那个版本



对于Level0 SSTable稍微特殊些, 因为对于这个级别SSTable files之间key会有重复的, 所以读的时候, 先找出level 0中哪些文件包含这个key, 并取最新的



image



怎样从.sst文件里面读到数据?



levelDb一般会先在内存中的Cache中查找是否包含这个文件的缓存记录，如果包含，则从缓存中读取；如果不包含，则打开SSTable文件，同时将这个文件的索引部分加载到内存中并放入Cache中。 这样Cache里面就有了这个SSTable的缓存项，但是只有索引部分在内存中，之后levelDb根据索引可以定位到哪个内容Block会包含这条key，从文件中读出这个Block的内容，在根据记录一一比较，如果找到则返回结果，如果没有找到，那么说明这个level的SSTable文件并不包含这个key，所以到下一级别的SSTable中去查找



 



可以看出对于SSTable, 相对写操作，读操作处理起来要复杂很多，所以写的速度必然要远远高于读数据的速度，也就是说，LevelDb比较适合写操作多于读操作的应用场合。而如果应用是很多读操作类型的，那么顺序读取效率会比较高，因为这样大部分内容都会在缓存中找到，尽可能避免大量的随机读取操作。

