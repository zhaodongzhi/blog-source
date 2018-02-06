---
title: 记mysqltest无法输出结果的一次调试经历
date: 2018-02-06 10:18:12
tags:
---

# 记mysqltest无法输出结果的一次调试经历

在一次测试中尝试用mysqltest对多条sql生成结果集，结果发现其中一条将会返回一个较大结果集(几个G)的查询迟迟不输出结果，但是用mysql客户端运行这条sql却可以很快输出结果

观察mysqltest卡住时的pstack

```c++
#0  0x00007fee0fc26a81 in memcpy () from /lib64/libc.so.6
#1  0x0000000000471616 in my_realloc ()
#2  0x0000000000476a22 in dynstr_append_mem ()
#3  0x0000000000424904 in replace_dynstr_append_mem(st_dynamic_string*, char const*, unsigned long) ()
#4  0x000000000041c30a in append_field(st_dynamic_string*, unsigned int, st_mysql_field*, char*, unsigned long, char) ()
#5  0x000000000041c43a in append_result(st_dynamic_string*, st_mysql_res*) ()
#6  0x000000000041d349 in run_query_normal(st_connection*, st_command*, int, char*, unsigned long, st_dynamic_string*, st_dynamic_string*) ()
#7  0x000000000041ea34 in run_query(st_connection*, st_command*, int) ()
#8  0x000000000042092d in main ()
```

可以看到mysqltest长时间处在memcpy处，同时通过htop会发现mysqltest占用的内存在以没几秒1M的速度上升。

我们再看一下函数dynstr_append_mem和my_realloc

```c++
my_bool dynstr_append_mem(DYNAMIC_STRING *str, const char *append,
              size_t length)
{
  char *new_ptr;
  if (str->length+length >= str->max_length)
  {
    size_t new_length=(str->length+length+str->alloc_increment)/
      str->alloc_increment;
    new_length*=str->alloc_increment;
    if (!(new_ptr=(char*) my_realloc(key_memory_DYNAMIC_STRING,
                                     str->str,new_length,MYF(MY_WME))))
      return TRUE;
    str->str=new_ptr;
    str->max_length=new_length;
  }
  memcpy(str->str + str->length,append,length);
  str->length+=length;
  str->str[str->length]=0;          /* Safety for C programs */
  return FALSE;
}

void *
my_realloc(PSI_memory_key key, void *ptr, size_t size, myf flags)
{   
  ...             
  new_ptr= my_malloc(key, size, flags);
  if (likely(new_ptr != NULL))
  {
    ...
    min_size= (old_size < size) ? old_size : size;
    memcpy(new_ptr, ptr, min_size);
    my_free(ptr); 
  
    return new_ptr;
  }
  return NULL;
}
```

可以发现当str内存不足时会重新申请内存，并将原str中的内容复制到新申请的内存中。注意到新申请的内存大小是

```c++
    size_t new_length=(str->length+length+str->alloc_increment)/
      str->alloc_increment;
    new_length*=str->alloc_increment;
```

那么当length和str->alloc_increment相差不大的时候，新申请的内存大小基本是几个str->alloc_increment上下，我们再来看一下main函数和append_result函数

```c++
int main(int argc, char **argv)
{
  ...
  while (!read_command(&command) && !abort_flag)
  {
    ...
    case Q_QUERY:
      ...
      run_query(cur_con, command, flags);
    ...
    /* Write result from command to log file immediately */
    log_file.write(&ds_res);
    log_file.flush();
  }
  ...
}

void append_result(DYNAMIC_STRING *ds, MYSQL_RES *res) 
{
  ...
  while ((row = mysql_fetch_row(res)))
  {
    uint i;
    lengths = mysql_fetch_lengths(res);
    for (i = 0; i < num_fields; i++)
    {
      /* looks ugly , but put here to convince parfait */
      assert(lengths);
      append_field(ds, i, &fields[i],
                   row[i], lengths[i], !row[i]);
    }
    if (!display_result_vertically)
      dynstr_append_mem(ds, "\n", 1);
  }
}
```

通过main函数我们可以看到mysqltest是将一条sql的结果处理完再从内存写入磁盘的，在处理sql结果的append_result函数中对于每一行数据都会调用append_field进而可能重新申请内存。通过gdb我们看到str->alloc_increment是2048，考虑到返回的结果集较大，且行数较多，而每一行数据基本不超过2048个字节，那么每append几行的时候可能就要重新申请内存，而且每次多申请的内存可能就只有一个str->alloc_increment大小，同时申请新内存也意味着要将之前的内容拷贝到新申请的内存中，结合htop中观察到的内存缓慢增长，可以认为mysqltest应该是由于每次新申请的内存过小而陷入了频繁的copy之前的行数据的循环中。

一个简单的解决方案：将alloc_increment做成一个可配置的变量，默认是2048，在返回结果集较大的情况下用户可以手动指定该值为一个较大的值，从而而避免频繁的内存拷贝导致长时间无法输出结果。

```c++
int main(int argc, char **argv)
{
  ...
  //init_dynamic_string(&ds_res, "", 2048, 2048);
  //init_dynamic_string(&ds_result, "", 1024, 1024);

  parse_args(argc, argv);

  init_dynamic_string(&ds_res, "", 2048, res_alloc_increment);
  init_dynamic_string(&ds_result, "", 1024, res_alloc_increment);
  ...
 }

void run_query(struct st_connection *cn, struct st_command *command, int flags)
{
  ...
  if (display_result_sorted)
  {
    /*
       Collect the query output in a separate string
       that can be sorted before it's added to the
       global result string
    */
    //init_dynamic_string(&ds_sorted, "", 1024, 1024);
    init_dynamic_string(&ds_sorted, "", 1024, res_alloc_increment);
    save_ds= ds; /* Remember original ds */
    ds= &ds_sorted;
  }
  ...
}
```

效果如下

```mysql
 ./mysqltest --help
   ...
   -I, --res-alloc-increment=# 
                      realloc increment bytes for result
```

本例中通过指定-I参数为1000000000成功使mysqltest快速输出了结果。如果小伙伴们有遇到类似mysqltest长时间无法输出结果的问题时可以考虑一下这个原因。