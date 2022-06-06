---
title: CMU 15-445 Project 2 Hash Index
date: 2022-05-15 21:25:32
tags:
  - 数据库
  - 公开课
  - CMU 15-445
categories:
  - 数据库
  - 公开课
---

# Overview

CMU 15-445 Fall 2021的第二个project是实现一个hash table，用以支持Bustub实现哈希索引。Project 2需要实现的hash table使用`extendible hashing scheme`也就是可扩展散列的技术。`Extendible hashing`是一种动态散列的模式，当数据库增大或缩小的时候，`extendible hashing`可以通过桶的分裂和合并来适应数据库大小的变化，每次重组仅作用于一个桶，避免了对整个结构全局频繁的重组操作，降低了性能开销。

和Project 1类似，我们需要实现预先提供的代码中声明的API，从而完成`extendible hashing`哈希索引的实现。

整个Project 2页拆分成了三个任务:

* **Page Layouts**
* **Extendible Hashing Implementation**
* **Concurrency Control**

第一个任务我们需要实现`Directory Page`和`Bucket Page`，`Directory Page`保存着哈希表的所有元数据，记录着`Bucket Page`相关的信息，而`Bucket Page`则保存着实际的数据，以及相关的元数据；第二个任务我们需要按照算法实现`extendible hashing`，实现包括查找、插入、删除等操作；第三个任务则需要在已实现`extendible hashing`的基础上，实现并发访问的控制。

# Page Layouts

我们需要实现的hash table需要通过数据库中的buffer Pool进行访问而不是通过我们自己分配内存来实现数据存取。Buffer Pool的操作是基于页的，所以我们需要实现`HashTableDirectoryPage`和`HashTableBucketPage`两个类来定义hash table的页。

## HashTableDirectoryPage

`HashTableDirectoryPage`类表示hash table的元数据，类中有以下属性：

```c++
/** page id **/
page_id_t page_id_;
/** 日志号，后面的project会用到**/
lsn_t lsn_;
/** 全局depth,depth表示使用hash的几个bit来定位数据，后面详细介绍 **/
uint32_t global_depth_{0};
/** 每个bucket对应的局部depth **/
uint8_t local_depths_[DIRECTORY_ARRAY_SIZE];
/** 所有bucket的page id **/
page_id_t bucket_page_ids_[DIRECTORY_ARRAY_SIZE];
```

`HashTableDirectoryPage`中主要提供一些简单的方法来维护这些元数据，比如`IncrGlobalDepth()`和`DecrGlobalDepth()`等，实现都比较简单，这里不再细说，只挑选几个我认为比较有用的helper方法讲讲作用和实现：

* `GetGlobalDepthMask`

  这个方法返回一个二进制表示为n位全1的uint32_t类型值，其中n等于`global_depth_`，把它与key的hash相与得到key所在的bucket的位置。实现为将1左移`global_depth_`位后减1。

* `GetSplitImageIndex(uint32_t bucket_idx)`

  bucket在数据已满的时候需要分裂一个新的bucket来保存数据，这个新的bucket所在的位置由这个方法来获得，其实现我们放到后面实现`extendible hashing`算法的时候来描述。

* `CanShrink()`

  这个方法用于判断当前hash table是否可以收缩，回收那些没有用到的bucket，其实现我们放到后面实现`extendible hashing`算法的时候来描述。

## HashTableBucketPage

`HashTableBucketPage`类是实际存储数据的地方，它有以下属性：

```c++
/** 如果occupied_中第i个位为1表示array_数组中第i个元素有插入过数据 **/
char occupied_[(BUCKET_ARRAY_SIZE - 1) / 8 + 1];
/** 如果readable_中第i个位为1表示array_数组中第i个元素有可读的数据 **/
char readable_[(BUCKET_ARRAY_SIZE - 1) / 8 + 1];
/** 零长数组，保存key-value的值**/
MappingType array_[0];
```

这里`occupied_`和`readable_`的区别需要注意下，如果在`array_`的一个位置上插入了一个key-value对，那相应的`occupied_`和`readable_`上对应位置均需要置为1，但是如果将该位置上的元素删除，只需要将`readable_`中对应位置置为0即可，`occupied_`保持为1就好。`occupied_`和`readable_`均使用1个bit来标记一个元素的状态，因此更新对应`array_`位置上元素的状态时需要通过一些位运算来找到`occupied_`和`readable_`上对应的bit。

Key-value对的插入和删除比较简单，插入前先遍历一遍看下待插入的key-value是否已存在，已存在返回false，否则就找到第一个`readable_`为0的位置插入即可。删除key-value对只需要遍历找到对应的位置，将该位置的`readable_`设为0即可。

第一个任务中我们只需要实现`HashTableDirectoryPage`中的`GetGlobalDepth` , `IncrGlobalDepth`,  `SetLocalDepth` , `SetBucketPageId`,  `GetBucketPageId`方法和`HashTableBucketPage`中的`Insert`, `Remove`, `IsOccupied`, `IsReadable`, `KeyAt`, `ValueAt`方法即可，不过在实现`extendible hashing`的过程中，我们可能会用到一些其他有用的方法，我们可以自己下来实现，比如上面讲到的`GetGlobalDepthMask`等，在下面实现`extendible hashing`算法的时候再来介绍。

还有一个需要注意的地方是，我们的`HashTableDirectoryPage`和`HashTableBucketPage`是从buffer Pool中获取来的，在实现中时直接使用`reinterpret_cast`从`Page`类型转换而来的，对应的就是`Page`类型头部的4k大小的`data_`，因此如非必要最好不要在`HashTableDirectoryPage`和`HashTableBucketPage`类中再添加额外的成员变量。`HashTableDirectoryPage`中因为`DIRECTORY_ARRAY_SIZE`定义为512所以4k的空间还算富裕，`HashTableBucketPage`中因为会保存大量的数据，可能会占用到4k的空间，如果再添加额外的属性可能导致访问越界，从而修改到`Page`类中的一些属性，导致问题的发生。

![page layout](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/page%20layout.png)

# Extendible Hashing Implementation

我们需要按照算法描述实现`extendible hashing`算法。`Extendible hashing`是一种dynamic hash的算法，通过动态扩展的方法来解决hash冲突。`extendible hash table`分为两部分，一部分称为directory，根据数据的hash值不同将数据分散到不同的directory上，另一部分称为bucket，是真正存储数据的部分，在同一个bucket上的数据hash值相同。

`Extendible hashing`中有一个`golbal depth`计数器，表示用数据的hash值的几位二进制位来定位该数据应该在哪个bucket之中。每个bucket上维护着自己的`local depth`，表示该bucket中的数据是用的几位hash值的二进制位来定位到这个bucket的。

![extendible hash](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/extendible%20hash.png)

如上图展示了一个`global depth`为2的`extendible hash table`，其中directory长度为4（directory长度总为`global depth`的2次幂）。总共有3个bucket，bucket中的数字表示的是bucket保存的数据的二进制hash值，根据hash值的LSB（最低有效位）来索引bucket，3个bucket的`local depth`分别为1、2、2，`local depth`总是小于等于`global depth`。

`Extendible hashtable`的查找就是简单地根据`global depth`和hash值定位bucket，然后遍历bucket查找即可。继续以上图为例，如果我们需要在hash table中查找`11011001`这个值，根据`global depth`为2，取`11011001`后两位`01`，定位到对应的bucket，找到了对应的数据。

`Extendible hashtable`插入数据时，也是先根据`global depth`和hash值定位需要插入的bucket，如果bucket未满，则直接将数据插入到bucket即可。如果bucket已满，则需要分裂bucket，也就是新开一个bucket用来保存数据。

![extendible hash-split](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/extendible%20hash-split.png)

如上图所示，假如我们要在这个hash table中插入hash值为`10100010`的数据，根据当前的`global depth`我们定位到`10`位置的bucket，我们可以看到`00`和`10`所指向的bucket为同一个bucket，原因我们后面分析，我们现在需要关注的是目前这个bucket中已经没有剩余的位置可以插入数据了，所以我们需要分裂bucket以便插入数据。我们先将这个bucket的`local depth`加1，然后将`10`位置指向一个新开的bucket，这个新开的bucket我们称为原来的bucket的`split image`，新bucket的`local depth`和原bucket加1后的`local depth`一样。分裂bucket完成之后我们需要将原bucket上的数据重新按`global depth`插入到相应的bucket中，然后再插入我们待插入的数据，这里`10100010`插入了`10`位置的bucket，也就是我们新开的那个bucket。

在分裂bucket的时候还有一种情况，即未分裂时bucket的`local depth`等于`global depth`时，我们需要扩展directory以索引更多的bucket。

![extendible hash-growth](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/extendible%20hash-growth.png)

如上图所示，当我们想要继续插入`11110110`到hash table中时，我们发现索引到的`10`位置的bucket已经满了并且`local depth`等于`global depth`，这时我们需要扩展directory的大小。`global depth`增加1，表示我们需要3个二进制位来索引bucket，directory的长度也应该扩展一倍。扩展后的directory可以索引更多的bucket，但是我们不会立即开辟新的bucket，而是将扩展部分的directory index指向已存在的bucket，`100`指向`000`所指向的bucket，`101`指向`001`所指向的bucket……以此类推，这样就完成了directory的扩展。接下来再按照分裂bucket的步骤进行插入即可。可以看到，directory发生扩展之后就会出现多个位置指向同一个bucket的情况，也就是上面我们留下的问题的答案。每次hash table扩容时，我们只需要重新处理局部的数据就可以完成扩容，而不需要重建整个hash table。

当删除bucket上的数据之后，如果bucket为空，我们可以合并bucket。合并bucket在某些`extendible hashing`上是没有被实现的，因为合并bucket可能在某些场景下产生性能抖动。当bucket为空，bucket与其`split image`的`local depth`相等且大于0时，bucket才可以与其`split image`合并。合并bucket就是将原本指向bucket的索引指向其`split image`，然后`local depth`减1，这样空bucket就不再存在于hash table之中。当hash table中合并的bucket越来越多时，很多不同的位置都是指向的同一个bucket，这时还可以进行directory的收缩，当且仅当所有bucket的`local depth`都小于`global depth`时，directory才可以进行收缩，收缩directory需要将`global depth`减1。

以上就是`extendible hashing`算法的描述，包括了查找、插入、删除、bucket的分裂和合并，directory的扩展和收缩等操作。在实现project时，如第一个任务所述，我们的directory和bucket是基于buffer pool来管理的，所以我们会用到`BufferPoolManagerInstance`类提供的API。当我们需要新开辟directory或bucket时，我们需要调用`NewPage`来申请一个新的页，并将返回的`Page`对象转换为`HashTableDirectoryPage`或`HashTableBucketPage`对象，使用一个已申请的页时需要调用`FetchPage`，当页使用完毕时调用`UnpinPage`取消页的固定，并根据页的变更状态正确设置`is_dirty` flag以保证写入的数据能够被安全地写入磁盘。

在实现这一任务的时候，我们可以不用考虑多线程并发时线程安全问题的处理，只需要实现单线程下的行为正确的`extendibel hashing`算法即可。

# Concurrency Control

在这一任务中我们需要实现并发安全的`extendibel hashing`算法。在实现上一任务的基础之上，我们需要使用锁来保护并发访问时的hash table。

代码中提供了两种粒度的锁，一种是定义于`ExtendibleHashTable`类中的`table_latch_`，锁的粒度为整个hash table实例，一种是`Page`类中的`rwlatch_`，锁的粒度为对应的`Page`对象，看代码的实现我们可以看到它们都是`ReaderWriterLatch`封装的读写锁，锁实现均为不可重入锁。

我们只有在需要对directory或bucket进行写入操作时才去获取对应粒度的写锁，其他时候一律只获取读锁。因为`global dept`、`local depth`、bucket page id等元数据都是在`HashTableDirectoryPage`类中维护的，因此我们可以等到存在数据操作时才对bucket进行加锁。还有一个点需要注意的是，当后续操作可能需要写锁时，就在操作最开始获取写锁，不要先获取读锁然后当需要写锁时才去获取写锁，因为代码中的锁是不可重入锁，获取另一个锁之前需要先释放持有的锁，这期间就有可能发生竞争，反而会产生并发问题，性能也不会得到优化。

# 总结

第二个project的难度相比第一个要大一点，主要是PPT中描述的`extendibel hashing`有些抽象导致理解多花费了些时间，选择采用LSB来定位bucket之后，实现的难度和理解起来其实会比PPT中使用MSB要简单些。

![image-1654531228246](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/image-1654531228246.png)

本次project实现的`extendibel hashing`可以作为实现数据库hash索引的基础。课程中还提到一种数据结构 - B+树，它也是实现数据库索引的一种优秀的数据结构，在2020学年的project 2是实现的B+树，我也准备尝试一下B+树的实现，以及B+树的并发安全访问实现。
