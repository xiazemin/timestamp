# LSM Tree

The Sorted String Table \(SSTable\) is one of the most popular outputs for storing, processing, and exchanging datasets. 

An SSTable is a simple abstraction to efficiently store large numbers of key-value pairs while optimizing for high throughput, sequential read/write workloads.



Unfortunately, the SSTable name itself has also been overloaded by the industry to refer to services that go well beyond just the sorted table, which has only added unnecessary confusion to what is a very simple and a useful data structure on its own. Let's take a closer look under the hood of an SSTable and how LevelDB makes use of it.



 



SSTable: Sorted String Table

SSTable本身是个简单而有用的数据结构, 而往往由于工业界对于它的overload, 导致大家的误解 

它本身就像他的名字一样, 就是a set of sorted key-value pairs 

如下图左, 当文件比较大的时候, 也可以建立key:offset的index, 用于快速分段定位, 但这个是可选的.





这个结构和普通的key-value pairs的区别, 可以support range query和random r/w



image



A "Sorted String Table" then is exactly what it sounds like, it is a file which contains a set of arbitrary, sorted key-value pairs inside. 

Duplicate keys are fine, there is no need for "padding" for keys or values, and keys and values are arbitrary blobs. Read in the entire file sequentially and you have a sorted index. Optionally, if the file is very large, we can also prepend, or create a standalone key:offset index for fast access.



That's all an SSTable is: very simple, but also a very useful way to exchange large, sorted data segments.



 



SSTables and Log Structured Merge Trees

仅仅SSTable数据结构本身仍然无法support高效的range query和random r/w的场景 

还需要一整套的机制来完成从memory sort, flush to disk, compaction以及快速读取……这样的一个完成的机制和架构称为,"The Log-Structured Merge-Tree" \(LSM Tree\) 

名字很形象, 首先是基于log的, 不断产生SSTable结构的log文件, 并且是需要不断merge以提高效率的



下图很好的描绘了LSM Tree的结构和大部分操作



image\_thumb\[3\]\[1\]  

We want to preserve the fast read access which SSTables give us, but we also want to support fast random writes. Turns out, we already have all the necessary pieces: random writes are fast when the SSTable is in memory \(let's call it MemTable\), and if the table is immutable then an on-disk SSTable is also fast to read from. Now let's introduce the following conventions:



 image



On-disk SSTable indexes are always loaded into memory

All writes go directly to the MemTable index

Reads check the MemTable first and then the SSTable indexes

Periodically, the MemTable is flushed to disk as an SSTable

Periodically, on-disk SSTables are "collapsed together"

What have we done here? Writes are always done in memory and hence are always fast. Once the MemTable reaches a certain size, it is flushed to disk as an immutable SSTable. However, we will maintain all the SSTable indexes in memory, which means that for any read we can check the MemTable first, and then walk the sequence of SSTable indexes to find our data. Turns out, we have just reinvented the "The Log-Structured Merge-Tree" \(LSM Tree\), described by Patrick O'Neil, and this is also the very mechanism behind "BigTable Tablets".



 



LSM & SSTables: Updates, Deletes and Maintenance

This "LSM" architecture provides a number of interesting behaviors: writes are always fast regardless of the size of dataset \(append-only\), and random reads are either served from memory or require a quick disk seek. However, what about updates and deletes?



Once the SSTable is on disk, it is immutable, hence updates and deletes can't touch the data. 

Instead, a more recent value is simply stored in MemTable in case of update, and a "tombstone" record \(不能直接删除,标上已deleted\) is appended for deletes. 

Because we check the indexes in sequence, future reads will find the updated or the tombstone record without ever reaching the older values! 

Finally, having hundreds of on-disk SSTables is also not a great idea, hence periodically we will run a process to merge the on-disk SSTables, at which time the update and delete records will overwrite and remove the older data.



 



SSTables and LevelDB

Take an SSTable, add a MemTable and apply a set of processing conventions and what you get is a nice database engine for certain type of workloads. 

In fact, Google's BigTable, Hadoop's HBase, and Cassandra amongst others are all using a variant or a direct copy of this very architecture.



Simple on the surface, but as usual, implementation details matter a great deal. Thankfully, Jeff Dean and Sanjay Ghemawat, the original contributors to the SSTable and BigTable infrastructure at Google released LevelDB earlier last year, which is more or less an exact replica of the architecture we've described above:



SSTable under the hood, MemTable for writes

Keys and values are arbitrary byte arrays

Support for Put, Get, Delete operations

Forward and backward iteration over data

Built-in Snappy compression

