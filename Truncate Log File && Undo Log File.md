# Truncate Log File && Undo Log File

标签（空格分隔）： 未分类

---
既然它调用了os_file_create，那么它肯定在存储层创建了一个文件，要不要对这个文件的写操作给屏蔽掉，直接用redo log来恢复呢
buf_page_write是一个要屏蔽的点
log_write_up_to是一个要改写的地方
truncate_t::write是否应该要屏蔽
能否通过redo log来恢复文件内容
##创建和删除时机
TruncateLogger对象负责管理文件的生命周期，上层操作truncate log file
TruncateLoggerParser解析truncate log file
truncate_t对象负责往文件中写入格式化的数据
###文件名
```
TruncateLogger::init(){
    m_log_file_name = srv_log_group_home_dir + s_log_prefix("ib_") + tablespace_id + table_id + s_log_ext("trunc.log");
}
```
##文件大小内容和格式
###文件大小
UNIV_PAGE_SIZE
###文件内容
|offset|length|content|
|---|---:|:---:|
|0|4|magic_n|
|4|8|lsn|
|12|4|space_id|
|16|4|format_flags|
|20|4|flags|
|24|2|tablename_len|
|26|tablename_len|tablename|
|||table|
||8|old_table_id|
||8|new_table_id|
||2|dir_len|
||dir_len|dir|
|||per index|
||8|index_id|
||4|index_type|
||4|index_root_page_no|
||4|index_trx_id|




##作用（读写）
崩溃恢复的时候，读到一条MLOG_TRUNCATE的redo log
```
truncate_t::parse_redo_entry
```
系统启动的时候
```
innobase_start_or_create_for_mysql
    TruncateLogParser::scan_and_parse(srv_log_group_home_dir)
        TruncateLogParser:scan      //扫描指定目录下的全部文件
        TruncateLogParser::parse
            os_file_create_simple
            os_file_read
            truncate_t::parse   //将解析得到的truncate_t对象添加到类truncate_t的静态变量中
            os_file_close
    recv_recovery_from_checkpoint_start
    trx_sys_init_db_start
    recv_apply_hashed_log_recs(TRUE)
    trx_purge_sys_create
    recv_recovery_from_checkpoint_finish
    truncate_t::fixup_tables_in_system_tablespace
    truncate_t::fixup_talbes_in_non_system_tablespace
    
```
```
truncate_t::fixup_tables_in_non_system_tablespace
    os_file_delete
```
写这个文件
```
TruncateLogger::log
    os_file_create
    m_truncate.write    //组织文件内容
    os_file_write
    os_file_flush
    os_file_close
    mlog_write_initial_log_record
    //写一条类型是MLOG_TRUNCATE的redo log
```
|内容|
|:---:|
|type(1)|
|space_id(4)|
|page_no = 0(4)|
|lsn(8)|


##truncate_t::tables_t truncate_t::s_tables

 1. 添加元素，TruncateLogParser::parse，解析完一个Truncate log file之后，生成一个truncate_t的对象，调用truncate_t::add(truncate)
 2. 删除元素，truncate_t::fixup_tables_in_system_tablespace,truncate_t::fixup_tables_in_no_system_tablespace
```
if(fsp_is_file_per_table){  //独立表空间
    if(!fil_space_get(space_id){
        fil_create_directory_for_tablename
        fil_ibd_create
    }
    fil_recreate_tablespace
}else{                      //共享表空间
    fil_recreate_table
}

row_truncate_update_sys_tables_during_fix_up
log_make_checkpoint_at
os_file_delete
```

##truncate_t::truncated_tables_t truncate_t::s_truncated_tables;

 1. **recv_parse_log_recs**解析redo log的时候遇上MLOG_TRUNCATE，解析redo log将space_id和lsn键值对添加到truncate_t::s_truncated_tables的哈希表中
 2. 当**recv_recover_page_func**将某一个page上的redo log完全应用的时候
```
if(srv_was_tablespace_truncated(fil_sace_get(recv_addr->space)){
    lsn_t init_lsn = truncate_t::get_truncate_tablesapce_init_lsn(recv_addr->space);
    skip_recv = (recv->start_lsn < init_lsn);
}
```
--------------------------------

#Undo Log File

##创建和删除时机
独立的undo表空间的truncate操作生成的log file
###文件名
```
namespace undo{
    dberr_t populate_log_file_name(
        ulint space_id,
        char * & log_file_name)
    {
        log_file_name = undo::s_log_prefix("undo_") + space_id + undo::s_log_ext("trun.log");
    }
    dberr_t init(ulint space_id){
        populate_log_file_name
        os_file_create
        os_file_write
        os_file_flush
        os_file_close
    }
    void done(ulint space_id){
        populate_log_file_name
        os_file_create_simple_no_error_handling
        os_file_write       //写入魔数
        os_file_flush
        os_file_close
        os_file_delete
    }
    bool is_log_present(ulint space_id){
        populate_log_file_name
        os_file_create_simple_no_error_handling
        os_file_status
        os_file_read    //检查魔数
        os_file_close
        os_file_delete
    }    
}
```
##文件大小内容和格式
###文件大小
UNIV_PAGE_SIZE
###文件内容

##作用（读写）
崩溃恢复的时候，读到一条MLOG_TRUNCATE的redo log