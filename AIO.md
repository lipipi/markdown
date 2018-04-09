# AIO

标签（空格分隔）： 未分类

---

##AIO的类型

 1. OS_AIO_NORMAL
 2. OS_AIO_IBUF
 3. OS_AIO_LOG
 4. OS_AIO_SYNC

##初始化
```
os_aio_init
    AIO::start
        s_reads = create
        s_ibuf = create
        s_log = create
        s_write = create
            AIO::init
                AIO::init_slots
//初始化四个AIO类的静态成员变量
```
四个AIO中所有的segment按序编号
per thread per segment
AIO::slot->size = n_pending_ios_per_thread * n_thread
|起始序号|数量|AIO|
|:---|---:|:---:|
|0|1|s_ibuf|
|1|1|i_log|
|2|n_read_thread|i_read|
|2+n_read_thread|n_write_thread|i_write|

任意一个io线程都能根据自己唯一的global_segment，首先确定是哪一个AIO，类型一定能相互匹配（get_array_and_local_segment)，然后确定是这个AIO中的哪一段slot
```
//每个槽存放了一个待执行的异步io任务
struct Slot{
    bool is_reserved;//ture表示这个槽中有IO任务待执行
    pfs_os_file_t file;//io的客体，相对应的文件
    fil_node_t * m1;
    IORequest type;//io操作的类型，OS_FILE_READ或者OS_FILE_WRITE
}
```
###收集异步io任务

>**<font color="red">AIO::reserve_slot</font>**

> 1. 检查有没有空闲的slot可以直接使用，如果没有挨个检查每个segment对应的slot，如果有待处理的io任务，就设置信号量，唤醒对应的io线程去处理
> 2. 尝试从对应的segment中找到一个空闲的slot，如果没有，就从AIO所有的槽中找出一个空闲的slot，可能上层有负载均衡的操作，将同类io任务均匀地分配到不同的io线程上，但是每个io线程执行任务的速度不一致，又会造成不均匀
> 3. 更新这个slot以及槽所在的AIO的状态，将slot返回

**IO负载均衡**
>local_segment = ( offset >> (UNIV_PAGE_SIZE_SHIFT + 6 )) % m_n_segment

<font color="blue">根据读写文件的偏移量来划分给各个线程（相同类型）</font>
这个线程的io任务太多了，32个slot放不下了，就将任务按序交给下一个线程