# os接口修改方案

-----
这个os函数被远程文件调用，可以在其中替换系统调用
否则不行
##计算层和存储层交互的文件
pfs_os_file_t包装起来的os_file_t == 日志文件 + 数据文件 ？？？
还有其他一些truncate log file，undo log file
是否需要写到存储层
崩溃恢复是否需要，多台计算节点，从机是否需要
```
RemoteDatafile::create_link_file
//直接通过fopen，fwrite，fclose对ISL文件进行操作（InnoDB Symbolic Link）
//ISL文件直接替换原本应该是ibd文件的位置
//文件内容，真实的ibd文件所在的路径
```

**基本的数据结构**
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
##<font color="SteelBlue">方案一：pfs_os_file_t</font>
###被其他模块调用的、相关的OS函数
**<ol>1.  使用open系统调用，打开文件，获取文件句柄，创建一个pfs_os_file_t对象</ol>**

 - Os_file_create_func
 - Os_file_create_simple_func
 - Os_file_create_simple_no_error_handling_func

**<ol>2.  检查文件是否支持sparse</ol>**
```
os_is_sparse_file_supported(pfs_os_file_t file)
    os_file_punch_hole(file.m_file,0,UNIV_PAGE_SIZE)
        os_file_punch_hole_posix(os_file_t file,off,len)
            return(DB_IO_NO_PUNCH_HOLE);
```
**<ol>3.  能否截断</ol>**
```
os_file_truncate(pfs_os_file_t file)
    os_file_get_size(file)
        os_file_truncate_posix(file)
            ftruncate(file.m_file)
```
**<ol>4.  获取或者设置文件大小</ol>**
*获取文件大小*
```
os_offset_t os_file_get_size(pfs_os_file_t file)
    os_offset_t pos = lseek(file.m_file,0,SEEK_CUR);//记录文件当前读写位置
    os_offset_t file_size = lseek(file.m_file,0,SEEK_END);//读取文件大小
    lseek(file.m_file,pos,SEEK_SET);//将指针复原到原始位置
    return(file_size);
```
*设置文件大小*
```
@param[in] type READ or WRITE
os_file_set_size(pfs_os_file_t file,ulint mode,type)
    os_aio
        pfs_os_aio_func
            os_aio_func
                
```
**<ol>5. 文件读写</ol>**
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
**<font color="red">优点：</font>**
偏上层，接口数量少，改动小，其他模块操作数据文件和日志文件都是通过pfs_os_file_t对象来和os模块交互的
**<font color="red">缺点：</font>**
 1. 部分os函数不依赖pfs模块只涉及os模块，可以直接迁移到存储层，但是部分os函数，*比如os_file_set_size*，涉及到pfs模块，难以完整迁移
 2. **pfs_os_file_t对象的生命周期**，pfs_os_file_t对象被缓存起来，在有些位置会出现，会跳过pfs_os_file，而直接访问文件句柄file.m_file的情况
 - 局部变量
 - AIO::reserve_slot，缓存在slot.file中
 - fil_node_t.handler
 - Datafile::m_handler

**<font color="red">**所以不能只修改os中和pfs_os_file_t有关的函数**</font>**

-----------
##<font color="SteelBlue">方案二：os_file_t</font>
直接从文件句柄入手
###os模块使用到的系统调用
```
fcntl.h
int open(const char * __file,int __oflag,...) __nonnull((1));
//根据文件名打开或者创建指定文件，返回文件句柄
int fcntl(int __fd,int __cmd,...);
//对这个文件执行cmd命令

unistd.h
int ftruncate(int __fd,__off_t __lenth) __THROW __wur;
//将一个已写入方式打开的文件截断
//返回0表示成功，返回-1表示失败

int fsync(int __fd）
//将内核中所有对该文件修改的缓存都刷新到磁盘

int close(int __fd);
//关闭文件句柄对应的文件
//返回0表示成功，返回-1表示失败

ssize_t pread(int __fd,void * __buf,size_t __nbytes,__off_t __offset) __wur;
//从文件指定偏移处读出指定数量的字符
ssize_t pwrite(int __fd,const void * __buf,size_t __n,__off_t __offset) __wur;
//往文件指定偏移处写入指定数量的字符

__off_t lseek(int __fd,__off_t __offset,int __whence) __THROW
//打开一个文件的时候，操作系统会自动为这个文件保留三个指针，0表示文件的起始位置，CUR表示当前读写的位置，END表示文件末尾
__whence有三种取值
 1. SEEK_CUR，返回从offset到文件当前读写位置的字节偏移量
 2. SEEK_END，返回从offset到文件末尾的字节偏移量，CUR指针会移动到文件末尾
 3. SEEK_CUR，将CUR指针移到offset指定的位置

```
**<font color="red">接口设计的原则：</font>**

 1. 简洁，避免复杂的参数和返回值
 2. 减少需要改动的位置
 3. 避免过多的网络交互，组合交互步骤
##被其他模块调用的、相关的OS函数
###<font color="DarkOliveGreen">使用文件名打开一个文件，返回文件句柄</font>
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

```flow
op1=>operation: os_file_create_simple_func
op2=>operation: os_file_create_func
op3=>operation: os_file_create_simple_no_error_hanlding_func
op4=>operation: os_file_get_status_posix
op5=>operation: os_file_close_func
op6=>operation: close
op7=>operation: os_file_get_status
op8=>operation: open
op1->op2->op3->op4->op5->op6->op7->op8
```


编写一个新的net_open函数，使用net_open替换掉，四个函数中对open的调用
还是重写调用了open的四个函数
选择前者
**<font color="Maroon">打开</font>**
```
//在以上四个函数中，使用net_open替换掉open
 - 计算层
int net_open( const char * file_name , int create_flag );

 - 存储层
ulint os_innodb_umask = S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP;
int net_open ( const char * file_name , int create_flag ){
    return open(file_name,create_flag,os_innodb_umask);
}
```
-----------
##<font color="DarkOliveGreen">使用文件id操作相应的文件</font>
**<font color="Maroon">1.   截断</font>**
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
**<font color="Maroon">2.   刷盘</font>**
![fsync_picture](https://raw.githubusercontent.com/lipipi/picture/master/fsync.png)
**函数堆栈**
```
os_file_flush_func ( os_file_t file )
    os_file_fsync_posix ( file )//涉及到线程休眠等
        fsync(file);
```
**修改方案**
```
//将fsync函数用net_fsync函数替换掉

 - 计算层
int net_fsync( int fd );

 - 存储层
int net_fsync( int fd ){
    return fsync(fd);
}

```
**<font color="Maroon">3.   关闭</font>**
**函数堆栈**
```
os_file_close_func ( os_file_t file )//涉及到错误信息收集
    close(file);
```
**修改方案**
```
//将close函数用net_close函数替换掉

 - 计算层
int net_close( int fd );

 - 存储层
int net_close( int fd ){
    return fsync(fd);
}
```
**<font color="Maroon">4.   读写</font>**
![read&write_picture](https://raw.githubusercontent.com/lipipi/picture/master/read%26write.png)
**函数堆栈**
```
os_file_write_func(os_file_t file)
    os_file_write_page(file)
        os_file_pwrite(file)
            os_file_io(file)

os_file_read_func(os_file_t file)
    os_file_read_page(file)
os_file_read_no_error_handling_func(os_file_t file)
    os_file_read_page(file)
os_file_read_page(os_file_t file)
    os_file_read(file)
        os_file_io(file)
        
os_file_io(os_file_t file,const IORequest & in_type)
    SyncFileIO sync_file_io(file)
    sync_file_io.execute
    sync_file_io_advance
    
SyncFileIO(os_file_t fh):m_fh(fh)//构造函数
SyncFileIO::execute(const IORequest & request)
    if(request.is_read())
        pread(m_fh);
    else
        pwrite(m_fh);

```
**修改方案**
**<font color="Maroon">5.   punch hole</font>**
**函数堆栈**
```
dberr_t os_file_io_complete (os_file_t file)//涉及到一些解压缩的操作
    os_file_punch_hole(file)
        os_file_punch_hole_posix(file)
            return(DB_IO_NO_PUNCH_HOLE);
            
AIOHandler::io_complete(const Slot * slot)
    os_file_io_complete(slot->file.m_file)

os_file_io(os_file_t file,const IORequest & in_type)
    os_file_io_complete
```
**修改方案**
**<font color="Maroon">其他系统调用都类似</font>**
**<font color="red">
大多数都是将系统调用层给替换掉
除了两处
</font>**
**<font color="Maroon"> 1. os_file_get_size</font>**
```
os_offset_t os_file_get_size(pfs_os_file_t file)
    os_offset_t pos = lseek(file.m_file,0,SEEK_CUR);//记录文件当前读写位置
    os_offset_t file_size = lseek(file.m_file,0,SEEK_END);//读取文件大小
    lseek(file.m_file,pos,SEEK_SET);//将指针复原到原始位置
    return(file_size);
```
进行了三次lseek系统调用
**修改方案**
```
//将这三次系统调用组合为一次网络交互

 - 计算层
os_offset_t net_file_get_size(os_file_t file);
os_offset_t os_file_get_size(pfs_os_file_t file){
    return net_file_get_size(file.m_file);
}

 - 存储层
os_offset_t net_file_get_size(os_file_t file){
    os_offset_t pos = lseek(file.m_file,0,SEEK_CUR);
    os_offset_t file_size = lseek(file.m_file,0,SEEK_END);
    lseek(file.m_file,pos,SEEK_SET);
    return(file_size);
}

```
**<font color="Maroon">2.   os_file_lock和os_file_set_nocache</font>**
```
int os_file_lock(int fd)
    struct flock lk;
    lk.l_type = F_WRLCK;
    lk.l_whence = SEEK_SET;
    lk.l_start = lk.l_len = 0;
    fcntl(fd,F_SETLK,&lk)

void os_file_set_nocache(int fd)
    fcntl(fd,F_SETFL,O_DIRECT)
```
调用的都是fcntl，如果使用net_fcntl将其替换掉，要传递三个参数，但是其中两个参数其实是确定的，避免传递过多的参数，所以直接设计两个接口
**调用时机**

 1. os_file_lock
 2. os_file_set_nocache
 
 ```
 os_file_create_func    --os0file.cc    //对数据文件和日志文件的操作
 row_merge_file_create  --row0merge.cc  //对临时文件的操作
 ```


**修改方案**
```

 - 计算层在两个函数内部将部分代码给替换掉
 
 - 存储层
int net_fcntl_file_lock(int fd){
    struct flock lk;
    lk.l_type = F_WRLCK;
    lk.l_whence = SEEK_SET;
    lk.l_start = lk.l_len = 0;
    return(fcntl(fd,F_SETLK,&lk));
}
int net_fcntl_file_set_nocache(int fd){
    return(fcntl(fd,F_SETFL,O_DIRECT));
}
```

**<font color="red">
相同的系统调用
相同的一套os_file_t
os_offset_t
os_innodb_umask
dberr_t
</font>**

------------------------
##<font color="SlateBlue">使用文件名访问</font>
###<font color="DarkOliveGreen">文件状态</font>
<font color="red">系统调用</font>
```
stat.h
extern int stat(const char * __restrict __file,struct stat * __restrict __buf) __THROW __nonnull ((2));
//获取文件信息
//返回0表示文件存在
```
**<font color="red">被调用的时机</font>**
```
//使用文件名检查文件状态，os_file_status在很多模块都被调用了
os_file_status_posix(const char * path)     //仅在os_file_status中被调用
    int ret = stat(path,&statinfo);
    根据stat_info.st_mode，设置type=OS_FILE_TYPE_DIR,OS_FILE_TYPE_LINK,OS_FILE_TYPE_FILE,OS_FIEL_TYPE_UNKNOWN
    return true if stat call succeeded
    
os_file_readdir_next_file(const char * path)
    int ret = stat(path,&statinfo);
    根据statinfo.st_mode，设置type=OS_FILE_TYPE_DIR,OS_FILE_TYPE_LINK,OS_FILE_TYPE_FILE,OS_FIEL_TYPE_UNKNOWN
    获取statinfo.st_size
    return true if stat call succeeded
    
os_file_get_size(const char * filename)
    int ret = stat(filename,&statinfo);
    获取statinfo.st_size和st_blocks

//使用文件名检查文件状态，os_file_get_status在很多模块都被调用了
os_file_get_status_posix(const char * path)     //仅在os_file_get_status中被调用
    int ret = stat(filename,&statinfo);
    根据statinfo.st_mode，设置type=OS_FILE_TYPE_DIR,OS_FILE_TYPE_LINK,OS_FILE_TYPE_FILE,OS_FIEL_TYPE_UNKNOWN
    获取statinfo.st_size,st_blksize,st_blocks
```

**<font color="Maroon">文件大小</font>**
```
struct os_file_size_t{
    os_offset_t m_total_size;
    os_offset_t m_alloc_size;
}
os_file_size_t os_file_get_size(const char * filename){
    struct stat s;
    os_file_size_t file_size;
    int ret = stat(filename,&s);
    if(ret == 0){
        file_size.m_total_size = s.st_size;
        file_size.m_alloc_size = s.st_blocks * 512;
    }else{
        file_size.m_total_size = ~0;
        file_size.m_alloc_size = (os_offset_t) errno;
    }
    return(file_size);
}
```
---------
###<font color="DarkOliveGreen">关闭文件</font>
<font color="red">系统调用</font>
```
unistd.h
extern int unlink ( const char * __name ) __THROW __nonnull ((1));
//
```
**<font color="red">被调用的时机</font>**
```
os_file_delete_if_exists_func(const char * name,bool * exist)
os_file_delete_func(const char * name)
```
-------------
###<font color="DarkOliveGreen">重命名文件</font>
<font color="red">系统调用</font>
```
stdio.h
extern int rename ( const char * __old , const char * __new ) __THROW;
//
```
**<font color="red">被调用的时机</font>**
```
os_file_rename_func(const char * oldpath,const char * newpath)
```
----------
#临时文件
##创建
通过mysql层创建文件
所以这个临时文件还是保留在本地
找出对临时文件的所有操作，不修改为远程文件操作
临时文件对象通常通过File对象保存在内存中，主要通过fXXX的系统调用来访问文件，不过也可以通过fileno(File)来获取os_file_t文件句柄对象，就可以调用os_XXX函数了，要找到这些地方，依旧访问本地文件，不访问远程文件
```
os_file_create_tmpfile(const char * path)
    int fd = innobase_mysql_tmpfile(path)
    fdopem(fd,"w+b")
```







