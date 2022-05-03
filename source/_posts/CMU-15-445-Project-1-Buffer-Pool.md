---
title: CMU 15-445 Project 1 Buffer Pool
date: 2022-05-02 19:26:10
tags:
  - 数据库
  - 公开课
  - CMU 15-445
categories:
  - 数据库
  - 公开课
---

# Overview

CMU 15-445 Fall 2021的课程期间需要完成一个基于磁盘的数据库Bustub，而这是其第一个Project（跳过了Project 0），其目标是构建一个Buffer Pool，Buffer Pool负责将物理页面从内存和磁盘之间进行换进换出。DBMS中执行引擎需要用到某页时，并非之间从磁盘中读取，而是从Buffer Pool中获取该页，由Buffer Pool负责页面的获取、淘汰。

![磁盘、Buffer Pool和执行引擎](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/image-20220502223257589.png)

<!--more-->

Buffer Pool内部维护一个固定大小的frame数组，记录Buffer Pool中缓存的页，还有一个page table用于记录page id与frame之间的关系。具体的Buffer Pool还需要通过具体的策略控制页面的淘汰，记录每个页面的使用情况以实现线程安全的控制、数据的安全等。

虽然看上去Buffer Pool的实现有些复杂，但是Project已经帮我们拆解好了任务，我们可以通过进行页面淘汰策略和页的管理两个方面的工作来实现Buffer Pool，具体到Project中分为以下三个任务:

* **LRU Replacement Policy**

* **Buffer Pool Manager Instance**

* **Parallel Buffer Pool Manager**

因为我学习的时候其实时看的2019年的课程视频，所以我还额外完成了19年的Project1中有的`Clock Replacement Policy`

# Clock Replacement Policy

Clock Replacement Policy是一种近似LRU的淘汰策略，它通过将页组织成一个环进行扫描，每个页上有一个reference bit用于记录该页是否被访问过，如果页面被访问则reference bit设为1，否则为0。

每次扫描时，如果页面的reference bit为1，说明页面自上次扫描之后有过访问，则仅将reference bit设为0不进行淘汰；如果reference bit设为0，说明页面自上次扫描之后还没有被访问过，就可以将页面进行淘汰了。

![Clock Replacement Policy描述](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/image-20220502232335875.png)

我们需要实现一个`ClockReplacer`，代码框架已经给出，需要按照给定的框架实现`Replacer`接口:

```c++
/*
 * 调用Victim方法表示需要淘汰一个Replacer中最近最少使用的frame，如果成功淘汰了这样的frame，将其frame id存储到frame_id中并返回true，
 * 否则返回false
 */
bool Victim(frame_id_t *frame_id);

/*
 * 调用Pin方法表示Buffer Pool需要使用该frame，将该frame从Replacer中删除
 */
void Pin(frame_id_t frame_id);

/*
 * 调用Unpin方法表示该frame没有线程在使用了，需要加入Replacer等待淘汰
 */
void Unpin(frame_id_t frame_id);

/*
 * 返回Replacer的大小
 */
size_t Size();
```

`ClockReplacer`的实现比较简单，需要注意记录`clock_hand`也就是当前扫描的clock位置、clock是否存在以及是否被访问，然后根据算法描述去更新这些信息就好了，注意加锁保证线程安全。

# LRU Replacement Policy

LRU应该很熟悉了，最近最少使用淘汰，可以看下[146. LRU 缓存 - LeetCode](https://leetcode-cn.com/problems/lru-cache/)。在Project中，我们需要实现一个`LRUReplacer`，还是按照上面的`Replacer`接口，这个接口和LeetCode上的get、put等还不太一样。

这里的`LRUReplacer`中记录的是可以被淘汰的页，而不是全部的页。Pin的过程是从`LRUReplacer`中删除该页面，这样该页面就不会被淘汰了。Unpin则是在`LRUReplacer`中加入该页面，这样该页面就可能被淘汰。

`LRUReplacer`我们还是通过hashmap+双向链表的形式来实现，链表中存储`frame_id`，每次将最近访问过的`frame_id`添加到链表头部，每次淘汰`frame_id`从链表尾部开始淘汰。hashmap中存储`frame_id`以及`frame_id`在链表中的位置，用于实现以O(1)时间复杂度获取到`frame_id`。具体的实现如下:

## bool Victim(frame_id_t *frame_id)

只要链表不为空，则从链表尾部淘汰一个`frame_id`返回，删除hashmap中对应的`frame_id`，并返回true，否则返回false。

## void Pin(frame_id_t frame_id)

在hashmap中找到`frame_id`，直接将该frame从hashmap和链表中都删除。

## void Unpin(frame_id_t frame_id)

当`frame_id`不在hashmap中且链表还未满时，把`frame_id`插入链表头部，hashmap记录`frame_id`在链表中的位置。

## size_t Size()

直接返回链表的size。

以上的实现也要注意加锁保证线程安全。

# Buffer Pool Manager Instance

`BufferPoolManagerInstance`即是管理Buffer Pool的实体，负责从`DiskManager`获取页面并将它们存储在内存中。 `BufferPoolManagerInstance`也可以将脏页写回磁盘以持久化数据变更。

在内存中，数据库页总是以`Page`对象来表示，在系统的整个生命周期中，相同的`Page`对象可能包含不同的物理页面，`Page`对象中我们需要关心的属性有:

```C++
/** page对象实际存储的数据 */
char data_[PAGE_SIZE]{};
/** page id */
page_id_t page_id_ = INVALID_PAGE_ID;
/** 表示page被多少线程使用，引用计数*/
int pin_count_ = 0;
/** 该page是否为脏页 */
bool is_dirty_ = false;
```

`BufferPoolManagerInstance`中我们需要关心的属性有:

```c++
/** page数组基地址，pages_+frame_id就可以定位到一个page对象 */
Page *pages_;
/** page table，记录page_id与frame_id之间的映射关系 */
std::unordered_map<page_id_t, frame_id_t> page_table_;
/** 负责淘汰页面，这里我们都使用LRUReplacer */
Replacer *replacer_;
/** 空闲frame_id列表 */
std::list<frame_id_t> free_list_;
```

`BufferPoolManagerInstance`中我们需要实现的接口有:

```c++
  /**
   * 从Buffer Pool中获取一个页面
   * @param page_id 需要获取的页的page id
   * @return 获取到的页面
   */
Page *FetchPgImp(page_id_t page_id);

  /**
   * 从Buffer Pool中unpin一个页面
   * @param page_id 需要unpin的page id
   * @param is_dirty 是否需要标记该页为脏页
   * @return 如果该页面的pin count在调用前已经小于等于0了就返回false，否则返回true
   */
bool UnpinPgImp(page_id_t page_id, bool is_dirty);

  /**
   * 将Buffer Pool中的指定页面写回磁盘
   * @param page_id id 需要写回的page id，不能是INVALID_PAGE_ID
   * @return 如果页面在page table中不存在则返回false，否则返回true
   */
bool FlushPgImp(page_id_t page_id);

  /**
   * 创建一个新的页面
   * @param[out] page_id 新页面的page id写入这个参数
   * @return 创建成功返回新创建的页的指针，否则返回null
   */
Page *NewPgImp(page_id_t *page_id);

  /**
   * 从Buffer Pool中删除一个页
   * @param page_id 待删除的页的page id
   * @return 删除成功返回true，失败返回false
   */
bool DeletePgImp(page_id_t page_id);

  /**
   * 把Buffer Pool中缓存的所有页面都写回磁盘
   */
void FlushAllPgsImp();
```

Buffer Pool、Replacer和Disk Manager的交互逻辑如下:

![buffer pool](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/buffer%20pool.png)

Replacer类似于一个页面回收站，维护了没有线程使用的可以被清除的frame的`frame_id`，Buffer Pool每次使用完一个页面时调用`UnpinPgImp`将页面放入回收站，在需要获取页面时可以从回收站获取可以被复用的frame的`frame_id`。Buffer Pool从磁盘读取页面数据通过调用Disk Manager的`ReadPage`方法实现，Buffer Pool将内存页面数据写回磁盘通过调用Disk Manager的`WritePage`方法实现。

frame在Buffer Pool和Replacer之间流转，一共有三种状态，分别是free、pinned和unpinned，frame的状态流转如下:

![frame status](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/frame%20status.png)

具体来讲一讲每个接口的实现逻辑和需要注意的细节:

## Page *BufferPoolManagerInstance::NewPgImp(page_id_t *page_id)

给出的代码中已有详细的逻辑注释，方法执行逻辑流程如下:

![new page](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/new%20page.png)

需要注意的几个点是:

* 先尝试从`free_list_`中获取空闲的frame，如果没有空闲的frame再尝试从`Replacer`获取一个淘汰的frame。
* 从`Replacer`中获取的淘汰的frame，一定要检查是否为脏页，如果为脏页需要先把先前的数据写回磁盘，以免脏页数据丢失。
* 从`Replacer`中获取的淘汰的frame，需要从`page_table_`中删除这个frame之前的`page_id_`与`frame_id`的映射。后面再用新的`page_id`映射`frame_id`。

## Page *BufferPoolManagerInstance::FetchPgImp(page_id_t page_id)

请求指定`page_id`的内容，返回`page`指针，同样在给出的代码框架中已有详细的逻辑注释了，逻辑流程如下:

![fetch page](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/fetch%20page.png)

需要注意的点:

* 如果`page_table_`中能找到对应的`page_id`，则可以不用读磁盘，直接返回对应的`page`指针。
* 如果`page_table_`中能找到对应的`page_id`，返回对应`page`指针之前，需要调用`replacer_`的`Pin`方法把对应的frame从回收站拿出来，`pin_count_`++。
* 更新`page`的metadata时，记得pin_count_++，表示线程正在使用这个页。

## bool BufferPoolManagerInstance::FlushPgImp(page_id_t page_id)

`FlushPgImp`主要操作就是调用`disk_manager_`的`WritePage`方法将页的内容写回磁盘，逻辑比较简单，只要`page_id`不等于`INVALID_PAGE_ID`即-1，且`page_table_`中存在对应`page_id`，就将页内容写回磁盘，顺便将页的`is_dirty_`置为false。

## void BufferPoolManagerInstance::FlushAllPgsImp()

这个接口的实现也比较简单，就是遍历`page_table_`，将里面的页的内容全部写回到磁盘，顺便将页的`is_dirty_`置为false。

## bool BufferPoolManagerInstance::DeletePgImp(page_id_t page_id)

这个接口可以直接删除Buffer Pool中的一个页，不用等`replacer_`去淘汰这个页。

具体的逻辑流程如下:

![delete page](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/delete%20page.png)

需要注意的点:

* 只有`pin_count_`不为0时才返回false，其余情况都是返回true，`page_table_`中找不到`page_id`时也返回true，这个注释里有说。
* 这里是直接删除页面，所以不用再调用`replacer_`的`Unpin`方法把frame加入到回收站再等待淘汰，删除页面后的`frame_id`直接加入`free_list_`。

##  bool BufferPoolManagerInstance::UnpinPgImp(page_id_t page_id, bool is_dirty)

这个接口代码中没有给出太多的注释，不过经过上面的状态流转分析以及数据流分析等，这个方法的逻辑应该也比较清楚，具体的逻辑流程如下:

![unpin page](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/unpin%20page.png)

需要注意的点:

* 只有调用时`pin_count_`已经小于等于0才返回false，其余情况都是返回true，`page_table_`中找不到`page_id`时也返回true，这个注释里有说。
* 不能直接将`page`的`is_dirty_`属性直接更新成参数`is_dirty`的值，因为可能出现`page`本身已经是脏页但是`is_dirty`给的是false，就有可能丢失数据。
* 只有`pin_count_`-1后为0时，才需要调用`replacer_`的`Unpin`方法。

所有的接口都要记得加锁来保证线程安全。

# Parallel Buffer Pool Manager

这是一个并行Buffer Pool管理器，内部管理着多个`BufferPoolManagerInstance`，对外暴露和`BufferPoolManagerInstance`同样的接口（它们都实现了`BufferPoolManager`接口）。

`ParallelBufferPoolManager`内部维护了几个属性:

```c++
// BufferPoolManagerInstance实例数
size_t num_instances_;
// 下一个轮询的BufferPoolManagerInstance实例的index，NewPgImp接口有用到
size_t next_instance_index_;
// 每个实例的大小
size_t pool_size_;
// 每个实例的指针
std::vector<BufferPoolManager *> instances_;
```

`ParallelBufferPoolManager`会将不同的`page_id`映射到不同的`BufferPoolManagerInstance`实例上进行管理，我们需要实现一个`GetBufferPoolManager`方法来完成这个映射，这里可以使用最简单的取模进行映射:

```c++
BufferPoolManager *ParallelBufferPoolManager::GetBufferPoolManager(page_id_t page_id) {
  // Get BufferPoolManager responsible for handling given page id. You can use this method in your other methods.
  return instances_[page_id % num_instances_];
}
```

其他的接口就是调用这个方法选择对应的`BufferPoolManagerInstance`实例，调用实例上的对应接口来执行，这里就不再赘述。

比较特殊的是`NewPgImp`接口的实现，代码给出了注释，要求我们从一个起始位置开始进行轮询，直到某一个实例`NewPgImp`成功返回`page`指针，或者再次回到起始位置，返回null。这里我们就用到了`next_instance_index_`。`next_instance_index_`的更新也是每次+1后再与`num_instances_`取模。

因为这是一个并行的Buffer Pool管理器，不同的`page_id`可以分散在不同的`BufferPoolManagerInstance`上进行管理，所以只有当操作的两个页同时在一个`BufferPoolManagerInstance`中时，才会有锁竞争，`ParallelBufferPoolManager`中可以不加锁。在`BufferPoolManagerInstance`中我们也可以看到`AllocatePage`方法分配新的`page_id`时，是会有一个`num_instances_`的步长进行自增的，这里的`num_instances_`就是从`ParallelBufferPoolManager`的`num_instances_`来的，这样也保证了`page_id`的不冲突。

# 总结

经过两天的查阅资料、讨论、不断尝试，终于在Gradescope上通过了Project1的测试。

![image-20220503002900620](https://images-1253386616.cos.ap-guangzhou.myqcloud.com/image-20220503002900620.png)

通过完成了这一个Project，把前面几讲课程的内容串起来了，不得不说对于数据库的认识又加深了，理解了Buffer Pool作为数据库管理内存的缓冲池，支撑起数据库中远大于内存容量的数据内容的工作原理。

在这个Project的实现中，我一是为了图方便，二是确实对应C++的内容不太熟悉，对于线程安全的保障只是简单的使用`std::lock_guard`来加解锁，这样虽然能通过评分测试，但是在实际的数据库应用中，这肯定是会带来性能隐患的，待后续来优化了。

其实完成之后回头看Buffer Pool的实现并没有一开始我们想象的那么复杂，尽管我们实现的只是一个简单版本的Buffer Pool，但是核心逻辑在梳理之后还是很清晰的。这其实不仅仅是因为我们实现的要求简单，还因为Project给出的代码框架有足够好的抽象和解耦，我们只需要明白每个接口的作用，梳理出Buffer Pool核心的数据流向和状态流转之后，就可以进行实现，这在系统设计层面也给我带来了不小的收获。

继续努力学习！

