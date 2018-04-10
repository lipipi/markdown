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

-----------------
##IO线程
默认一个ibuf，一个log，四个read，四个write线程
每个线程分配一个全局的segment号，从0开始，依次递增1
等待在各自的segment号上
```
//fil0fil.cc
fil_aio_wait(segment)
    //os0file.cc
    os_aio_handler(segment,&fil_node,&message,&type)
        os_aio_simulated_handler(segment,&fil_node,&message,&type)
            1.首先根据全局的segment找到是哪一个AIO和对应的local_segment
            2.SimulatedAIOHandler handler对象，检查该segment对应的32个slot中，找到一个io完成了的slot
            3.AIO将该slot释放掉
    fil_node_complete_io
    buf_page_io_complete
    log_io_complete
    
```
>需要存储到存储层的文件
>都是通过os函数来访问
>不是所有文件都是通过os函数来访问的
>不需要存储到存储层，由mysql创建的临时文件，也调用了os函数
>os函数如何区分该文件是本地的，还是远程的
>不能区分的话，复制一套net_os
>net_os和os函数基本一致，只是将里面的系统调用替换为网络交互
>然后找出所有的远程文件，将访问这些文件的os函数用net_os函数替换掉

消息格式
```
struct param{
    int param_type,
    int param_len,
    char value[param_len]
}
struct request{
    int type,
    int param_num,
    param params[param_num],
    int return_num,
    int return_types[return_num]
}
struct response{
    int return_num,
    param returns[return_num]
}
```
数据文件和日志文件
pfs_os_file fil_node->handler
pfs_os_file datafile->m_handler
都是通过文件句柄来访问的
但是其他文件比如truncate log file和undo log file没有缓存pfs_os_file
都是通过文件名来访问的

##计算层和存储层交互
使用socket通信
计算层：
异步io任务，由io线程在执行
同步io任务，由用户线程执行
```
class SocketPool{
static{
    WSADATA wsd;
    if(WSAStartup(MAKEWORD(2，2),&wsd) != 0){
        ib:error()<<"WSAStartup failed!";
    }
}
public:
    static SocketPool * getInstance();
    SOCKET getSocket(bool = false);
    void returnSocket(SOCKET socket);
    ~SocketPool();
private:
    SocketPool(const char * addr,short port,int size):size(size):n_reserved(size){
        servAddr.sin_family = AF_INET;
        servAddr.sin_addr.s_addr = inet_addr(addr);
        servAddr.sin_port = htons(port);
        sockets = new sockets[size];
        for(int i = 0;i<size;i++){
            sockets[i] = socket(AF_INET,SOCKET_STRAM,IPPROTO_TCP);
        }
    }
public:
private:
    int size;
    SOCKET * sockets;
    SOCKADDR_IN servAddr;
    int n_reserved;
```