# MINISQL实验报告
!!! abstract "进度"
    - [x] #1 Disk and Buffer pool manager
    - [ ] #2 Record manager
        * [x] serialization and deserialization
        * [x] implement `TableHeap` and `TableIterator`
    - [ ] #3 Index manager
        * [ ] Internal page
        * [ ] leaf page
        * [ ] B plus tree structure
        * [ ] index iterator
    - [ ] #4 Catalog manager
    - [ ] #5 Planner and Executor

## #2 Record manager
### `TableHeap`
`TableHeap`是以**无序**方式存储一张表中的记录的。它的核心就在于对堆表中的每个数据页进行操作，进行插入、删除、更新记录。因此有必要记录一下`TablePage`的各个函数功能

+ `bool TablePage::UpdateTuple(...)`
更新一条记录。`old_row`中存有该记录的`RowId`，由此可以定位到该记录。

首先检查这条记录是否被删除；然后检测剩余空闲空间+该记录大小是否能大于新记录；

然后用`memmove`函数，将从`free_space_pointer`到 ==该记录之前== 的所有数据进行移动，不妨理解为`tuple_size < serialized_size`，那么就需要为新记录腾出空间，数据向前移动`serialized_size - tuple_size`。

然后更新`free_space_pointer`，将新记录序列化到腾出的空间中，更新记录的大小；

最后把修改之前所有offset小于等于该记录的记录的offset都更新（减去`serialized_size - tuple_size`）

==但是== ，该函数需要修改，因为若返回false，并不知道具体的更新失败原因，我的做法是改为int返回值

!!! warning 
    `TableHeap`中的所有函数都要用到某个page，注意函数结束时unpin该page

## #3 Index manager
### Internal Page
对于各函数作用的解释
#### Helper function
+ `InternalPage::Init`
设置自己的page_id, parent_page_id

+ `KeyAt(int index)`/`SetKeyAt(int index)`

+ `ValueAt(int index)`/`SetValueAt(int index)`

+ `int ValueIndex(const page_id_t &value)`
根据pageId，也就是value来找对应的index

+ `void *PairPtrAt(int index)`
根据index，返回对应的pair（key）的地址

+ `void PairCopy(void *dest, void *src, int pair_num)`
复制`pair_num`个pair到dest

#### LoopUp
+ `page_id_t Lookup(const GenericKey *key, const KeyManager &KM) `
根据`key`二分查找，并返回对应的页码（value）

#### Insertion
+ `void PopulateNewRoot(const page_id_t &old_value, GenericKey *new_key, const page_id_t &new_value)`
new一个page作为新的root page，用`new_key`和`new_value`给它添加一个pair；自己的value更新为`new_value`，root page的value设置为`old_value`

+ `int InsertNodeAfter(const page_id_t &old_value, GenericKey *new_key, const page_id_t &new_value)`
在`old_value`对应的pair后面插入一个新pair，更新size

#### Split
+ `void MoveHalfTo(InternalPage *recipient, BufferPoolManager *buffer_pool_manager)`
将一半的pair移到recipient，然后更改这些pair的父亲页码

> 此处的recipient应该是新构造的空页。但是第一个key应该怎么设置？

+ `void CopyNFrom(void *src, int size, BufferPoolManager *buffer_pool_manager)`
将n个pair添加到自身的末尾，并将他们的父亲设置为自身

#### Remove
+ `void Remove(int index)`
删除index处的pair

+ `page_id_t RemoveAndReturnOnlyChild()`
删除唯一的孩子

#### Merge
+ `void MoveAllTo(InternalPage *recipient, GenericKey *middle_key, BufferPoolManager *buffer_pool_manager)`
将自身所有的pair移到recipient，同时更新它们的父亲；在自己的parent page中删除自己对应的pair。调整size

!!! note
    转移时别忘了将自己parent中对应的key也转移到recipient中，作为第一个append的key，也即`middle_key`应获取的值。这样key和value的数量才配对。

#### Redistribute
+ `void MoveFirstToEndOf(InternalPage *recipient, GenericKey *middle_key, BufferPoolManager *buffer_pool_manager)`
将自身的第一个pair移到recipient末尾，同上，不能忘记`middle_key`的转移。同时，更新pair的父亲。同时，更新自己父亲对应的key，因为自己最小的key变化了。

+ `void CopyLastFrom(GenericKey *key, const page_id_t value, BufferPoolManager *buffer_pool_manager)`
将pair加到末尾，将它的父亲更新为自己

+ `void MoveLastToFrontOf(InternalPage *recipient, GenericKey *middle_key, BufferPoolManager *buffer_pool_manager)`
recipient所有pair右移，腾出位置用来存放recipient的parent对应的key（即recipient所在子树中的最小key），以及原先dummy head的value。然后将自身的最后一个pair移到recipient的开头，即将该pair的value赋给dummy head的value。

+ `void CopyFirstFrom(const page_id_t value, BufferPoolManager *buffer_pool_manager)`

