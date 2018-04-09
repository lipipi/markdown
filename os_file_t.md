# os_file_t

标签（空格分隔）： 文件

---
```
typedef int os_file_t;
typedef ib_uint64_t os_offset_t;
struct pfs_os_file_t
{
    os_file_t m_file;
    #ifdef UNIV_PFS_IO
        struct PSI_file * m_psi;
    #endif
};
```
系统调用
```
fcntl.h
int open(const char * __file,int __oflag,...) __nonnull((1));
//根据文件名打开或者创建指定文件，返回文件句柄

unistd.h
int ftruncate(int __fd,__off_t __lenth) __THROW __wur;
//将一个已写入方式打开的文件截断
//返回0表示成功，返回-1表示失败

int fsync(int __fd）
//将内核中所有对该文件修改的缓存都刷新到磁盘

int close(int __fd);
//关闭文件句柄对应的文件
//返回0表示成功，返回-1表示失败

__off_t lseek(int __fd,__off_t __offset,int __whence) __THROW
//打开一个文件的时候，操作系统会自动为这个文件保留三个指针，0表示文件的起始位置，CUR表示当前读写的位置，END表示文件末尾
__whence有三种取值
 1. SEEK_CUR，返回从offset到文件当前读写位置的字节偏移量
 2. SEEK_END，返回从offset到文件末尾的字节偏移量，CUR指针会移动到文件末尾
 3. SEEK_CUR，将CUR指针移到offset指定的位置

```
---------------
-------------
要改的
系统调用
open
ftruncate
写net_open函数，用net_open替换掉，调用open的四个函数
还是重写调用了open的四个函数
接口的参数、返回值，简洁性
减少交互的次数，有些基本的系统调用可以组合起来的，就组合起来，再往上层走
os_file_t
os_file_punch_hole
os_file_get_size

---------
---------
##os0file.cc中针对pfs_os_file_t对象的操作，会被其他模块调用的函数
**部分os函数不依赖pfs模块只涉及os模块**
**部分os函数涉及pfs模块，pfs模块又调用os的函数**
        *比如os_file_set_size*

###打开的pfs_os_file_t对象的生命周期

 - 局部变量
 - AIO::reserve_slot，缓存在slot.file中
 - fil_node_t.handler
 - Datafile::m_handler
**<font color="red">pfs_os_file_t对象被缓存起来，有些位置会出现，直接访问file.m_file的情况</font>**
**所以不能只修改os中和pfs_os_file_t有关的函数**
-----------
 1. 使用**open系统调用**，打开文件，获取文件句柄，创建一个pfs_os_file_t对象
<ol>Os_file_create_func</ol>
<ol>Os_file_create_simple_func</ol>
<ol>Os_file_create_simple_no_error_handling_func</ol>
-------

 2. 检查文件是否支持sparse
```
os_is_sparse_file_supported(pfs_os_file_t file)
    os_file_punch_hole(file.m_file,0,UNIV_PAGE_SIZE)
        os_file_punch_hole_posix(os_file_t file,off,len)
            return(DB_IO_NO_PUNCH_HOLE);
```
--------

 3. 能否truncate
```
os_file_truncate(pfs_os_file_t file)
    os_file_get_size(file)
        os_file_truncate_posix(file)
            ftruncate(file.m_file)
```

-------
 4. 获取或者设置文件大小
**获取文件大小**
```
typedef ib_uint64_t os_offset_t
os_offset_t os_file_get_size(pfs_os_file_t file)
    os_offset_t pos = lseek(file.m_file,0,SEEK_CUR);//记录文件当前读写位置
    os_offset_t file_size = lseek(file.m_file,0,SEEK_END);//读取文件大小
    lseek(file.m_file,pos,SEEK_SET);//将指针复原到原始位置
    return(file_size);
```
**设置文件大小**
```
@param[in] type READ or WRITE
os_file_set_size(pfs_os_file_t file,ulint mode,type)
    os_aio
        pfs_os_aio_func
            os_aio_func
                
```
 5. 文件读写
```
os_aio_func(pfs_os_file_t file , ulint mode , IORequest & type)
    if(mode == OS_AIO_SYNC){
        if(type == READ)
            os_file_read_func(file.m_file);
        else
            os_file_write_func(file.m_file);
    else{
        AIO * array = AIO::select_slot_arry(type);
        array->reserve_slot(file);
    }
```

##os0file.cc中针对os_file_t对象的操作，会被其他模块调用的函数
##使用文件名打开一个文件，获取文件id
```
ulint os_innodb_umask = S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP;
int open ( const char * , int , ulint );

pfs_os_file_t file;
file.m_file == ::open(name,create_flag,os_innodb_umask);
//将文件id封装在pfs_os_file_t对象中返回
```
**被调用的位置**
 - os_file_create_simple_func
 - os_file_create_func
 - os_file_create_simple_no_error_hanlding_func
 - os_file_get_status_posix
```
//在以上四个函数中，使用net_open替换掉open
 - 计算层
int net_open( const char * file_name , int create_flag );

 - 存储层
ulint os_innodb_umask = S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP;
int net_open_store ( const char * file_name , int create_flag ){
    return open(file_name,create_flag,os_innodb_umask);
}
```
-----------
##使用文件id操作相应的文件
**<font color="blue">ftruncate</font>**
```
int ftruncate ( int fd , int );

int res = ftruncate(file.m_file,size);
if(res == 1)
    输出错误信息
return res == 0;
```
**被调用的位置**

 - os_file_truncate_posix(pfs_os_file_t file)
 - os_file_set_eof(FILE * file)，*innodb对临时文件，错误输出文件等其他文件的操作
```
//只需要用net_ftruncate替换第一个函数中的ftruncate

 - 计算层
 int net_ftruncate( int fd , int size );
 
 - 存储层
 int net_ftruncate ( int fd , int size ){
    return ftruncate(fd,size);
 }
```
```
//将fsync函数用net_fsync函数替换掉

 - 计算层
int net_fsync( int fd );

 - 存储层
int net_fsync( int fd ){
    return fsync(fd);
}

```
```
//将close函数用net_close函数替换掉

 - 计算层
int net_close( int fd );

 - 存储层
int net_close( int fd ){
    return fsync(fd);
}

```

<font color="red">
相同的系统调用
相同的一套os_file_t os_offset_t
os_innodb_umask
dberr_t
</font>

**<font color="blue">os_file_punch_hole</font>**
```
dberr_t os_file_punch_hole ( os_file_t fh , os_offset_t off , os_offset_t len )
dberr_t os_file_io_complete ( const )//涉及到，，，
    os_file_punch_hole
SyncFileIO(os_file_t fh)构造函数
dberr_t os_file_punch_hole_posix ( os_file_t fh , os_offset_t off , os_offset_t len )
    return(DB_IO_NO_PUNCH_HOLE);
os_file_flush_func ( os_file_t file )
    os_file_fsync_posix ( file )//涉及到线程休眠等
        fsync(file);
os_file_close_func ( os_file_t file )//涉及到错误信息收集
    close(file);
os_file_io(os_file_t file,const IORequest & in_type)
    SyncFileIO sync_file_io(file)
    sync_file_io.execute
    sync_file_io_advance
    os_file_io_complete
os_file_pwrite(os_file_t file)
    os_file_io
```
---------



###os_file_lock(os_file_t,const char *)
###os_file_set_nocache(os_file_t,const char *,const char *)
#os_file_punch_hole(os_file_t,os_offset_t,os_offset_t)
能不能不要这个函数了，屏蔽掉直接上面两个少实现fcntl
不用走太底层了把，bingo
设计原则，接口简单
存储层需要执行的操纵，我要清晰
###lseek(os_file_t,int,int)
###ftruncate(os_file_t,int)
###os_file_punch_hole(os_file_t,int,int)
###pread(os_file_t,
###pwrite(os_file_t,
###lseek(
##关闭文件

仅在os0file.cc文件中被调用
在文件中添加代码
```
#define open
#ifndef __MA_NET_IO_
extern int open ( const char * __file , int __oflag , ...) __nonnull ((1));
#else
int net_open ( const char * __file , int __oflag ,ulint mask);
#endif
```





