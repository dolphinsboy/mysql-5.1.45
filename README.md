### 一. 构建新的存储引擎

#### 1. 新的存储引擎名称为spartan

```sql
cd storage 
make spartan
cp ./example  ./spartan
cd spartan
sed -i "s#example#spartan#g" *
sed -i "s#EXAMPLE#SPARTAN#g" *

#
sh BUILD/autorun.sh
./configure --prefix=/home/guosong/mysql5145 \
 --with-mysqld-user=mysql --with-debug \
 --with-plugins=archive,blackhole,innobase,myisam,example,spartan

make
make install

#在MySQL执行
INSTALL PLUGIN spartan SONAME 'ha_spartan.so';

```

**提示**

```
161103 19:52:43 [ERROR] Can't open shared library '/home/guosong/mysql5145/lib/mysql/plugin/ha_spartan.so' (errno: 0 undefined symbol: _ZN13Spartan_index10insert_keyEP9SDE_INDEXb)
```

>
在修改Makefile.in以及Makefile.am之后需要运行一下sh BUILD/autorun.sh，生成新的Makefile.in。
例如在实际操作中先添加spartan_data，后再添加spartan_index，中间需要运行sh BUILD/autorun.sh，否则lib找不到对应的spartan_index相关函数。


**在MySQL5.6中，修改操作非常简单：
修改CMakeList.txt文件即可。

**静态编译的模式**
```
SET(SPARTAN_PLUGIN_STATIC "ha_spartan")
SET(SPARTAN_PLUGIN_MANDATORY TRUE)
SET(SPARTAN_SOURCES
    ha_spartan.cc ha_spartan.h
    spartan_data.cc spartan_data.h
    spartan_index.cc spartan_index.h
    )
MYSQL_ADD_PLUGIN(SPARTAN ${SPARTAN_SOURCES} STORAGE_ENGINE MANDATORY)
```
#### 2.增加数据操作

编辑ha_spartan.h文件,添加如下内容:

```c
#include "spartan_data.h"
```

修改Makefile.am文件,添加如下内容:

```c
noinst_HEADERS =    ha_spartan.h spartan_data.h 
libspartan_a_SOURCES=   ha_spartan.cc spartan_data.cc

```

修改Makefile.in,添加如下内容:

```c
#BEGIN GUOSONG MODIFICATION
noinst_HEADERS = ha_spartan.h spartan_data.h
#END GUOSONG MODIFICATION

#BEGIN GUOSONG MODIFICATION
libspartan_a_SOURCES = ha_spartan.cc spartan_data.cc
#END GUOSONG MODIFICATION 

#BEGIN GUOSONG MODIFICATION
am_libspartan_a_OBJECTS = libspartan_a-ha_spartan.$(OBJEXT) spartan_data.$(OBJEXT)
#END GUOSONG MODIFICATION

```

修改CMakeLists.txt:

```c
SET(SPARTAN_SOURCES ha_spartan.cc spartan_data.cc)
```

修改configure文件,添加如下内容:

```c
待补充
```

目前通过INSTALL PLUGIN方式,这种方式的结果如下:

```sql

mysql> show plugins;
+------------+--------+----------------+---------------+---------+
| Name       | Status | Type           | Library       | License |
+------------+--------+----------------+---------------+---------+
| binlog     | ACTIVE | STORAGE ENGINE | NULL          | GPL     |
| ARCHIVE    | ACTIVE | STORAGE ENGINE | NULL          | GPL     |
| BLACKHOLE  | ACTIVE | STORAGE ENGINE | NULL          | GPL     |
| CSV        | ACTIVE | STORAGE ENGINE | NULL          | GPL     |
| MEMORY     | ACTIVE | STORAGE ENGINE | NULL          | GPL     |
| InnoDB     | ACTIVE | STORAGE ENGINE | NULL          | GPL     |
| MyISAM     | ACTIVE | STORAGE ENGINE | NULL          | GPL     |
| MRG_MYISAM | ACTIVE | STORAGE ENGINE | NULL          | GPL     |
| EXAMPLE    | ACTIVE | STORAGE ENGINE | ha_example.so | GPL     |
| SPARTAN    | ACTIVE | STORAGE ENGINE | ha_spartan.so | GPL     |
+------------+--------+----------------+---------------+---------+
10 rows in set (0.00 sec)
```

#### 3.将spatan存储引起添加到服务器

修改my_config.h,添加如下内容:
 
```c
/*BEGIN GUOSONG MODIFICATION*/
#define WITH_SPARTAN_STORAGE_ENGINE 1
/*END GUOSONG MODIFICATION*/

```

修改handler.h,添加如下内容:

```c
enum legacy_db_type
{
    ...
  DB_TYPE_SPARTAN_DB,
  DB_TYPE_FIRST_DYNAMIC=42,
  DB_TYPE_DEFAULT=127 // Must be last
    ...
}
```


#### 4.处理表操作

修改ha_spartan.h文件

```c
/** @brief
  SPARTAN_SHARE is a structure that will be shared among all open handlers.
  This spartan implements the minimum of what you will probably need.
*/
typedef struct st_spartan_share {
  char *table_name;
  uint table_name_length,use_count;
  pthread_mutex_t mutex;
  THR_LOCK lock;
  /*BEGIN GUOSONG MODIFICATION*/
  Spartan_data *data_class;
  /*END GUOSONG MODIFICATION*/

} SPARTAN_SHARE;
```

修改ha_spartan.cc的get_share方法:

```c
    /*BEGIN GUOSONG MODIFICATION*/
    /*Reason this Modification:
    ¦* create an instance of data class
    ¦*/
    share->data_class = new Spartan_data();
    /*END GUOSONG MODIFICATION*/ 
```

修改ha_spartan.cc文件里的free_share方法:

```c
    /*BEGIN GUOSONG MODIFICATION*/
    if(share->data_class != NULL)
    ¦   delete share->data_class;
    share->data_class = NULL;
    /*END GUOSONG MODIFICATION*/
```

设置数据文件和索引文件的后缀名称:

```c
/*BEGIN GUOSONG MODIFICATION*/
static const char *ha_spartan_exts[] = {
  SDE_EXT,
  SDI_EXT,
  NullS
};
/*END GUOSONG MODIFICATION*/
```

修改ha_spartan.cc中的create操作:

```c
  /*BEGIN GUOSONG MODIFICATION*/
  char name_buff[FN_REFLEN];

  if(!(share = get_share(name, table)))
      DBUG_RETURN(1);

  if(share->data_class->create_table(fn_format(name_buff, name, "",SDE_EXT,
                  MY_REPLACE_EXT|MY_UNPACK_FILENAME)))
        DBUG_RETURN(-1);
  share->data_class->close_table();
  /*END GUOSONG MODIFICATION*/

```

修改ha_spartan.cc中的delete_table操作:

```c
  /*BEGIN GUOSONG MODIFICATION*/
  char name_buff[FN_REFLEN];
  
  if(!(share=get_share(name, table)))
    ¦   DBUG_RETURN(1);
  
  pthread_mutex_lock(&spartan_mutex);
  share->data_class->close_table();
  my_delete(fn_format(name_buff, name, "",
    ¦   ¦ SDE_EXT,MY_REPLACE_EXT|MY_UNPACK_FILENAME), MYF(0));
  pthread_mutex_unlock(&spartan_mutex);
  /*END GUOSONG MODIFICATION*/
```

修改ha_spartan.cc中的rename_table操作:

```c
  /*BEGIN GUOSONG MODIFICATION*/
  char data_from[FN_REFLEN];
  char data_to[FN_REFLEN];
  
  if(!(share=get_share(from,table)))
    ¦   DBUG_RETURN(1);
  
  pthread_mutex_lock(&spartan_mutex);
  share->data_class->close_table();
  my_copy(fn_format(data_from, from, "",
    ¦   ¦ SDE_EXT,MY_REPLACE_EXT|MY_UNPACK_FILENAME),
    ¦   ¦ fn_format(data_to, to,"",SDE_EXT,MY_REPLACE_EXT|MY_UNPACK_FILENAME),
    ¦   ¦ MYF(0));
  share->data_class->open_table(data_to);
  pthread_mutex_unlock(&spartan_mutex);
  my_delete(data_from, MYF(0));
  DBUG_RETURN(0);
  /*END GUOSONG MODIFICATION*/
```

修改ha_spartan.cc中的open操作:

```c
int ha_spartan::open(const char *name, int mode, uint test_if_locked) 
{
  DBUG_ENTER("ha_spartan::open");

  /*BEGIN GUOSONG MODIFICATION*/ 
  char name_buff[FN_REFLEN];
  if (!(share = get_share(name, table)))
    DBUG_RETURN(1);
  share->data_class->open_table(fn_format(name_buff, name, "", SDE_EXT,
    ¦   ¦   ¦ MY_REPLACE_EXT|MY_UNPACK_FILENAME));
  /*END GUOSONG MODIFICATION*/
  thr_lock_data_init(&share->lock,&lock,NULL);

  DBUG_RETURN(0);             
}
```

make的时候遇到错误，修改Makefile，按照如下注释：

```c
do_abi_check:
    #set -ex; \
    #for file in $(abi_headers); do \
    #         gcc -E -nostdinc -dI \
    #                  -I$(top_srcdir)/include \
    #                  -I$(top_srcdir)/include/mysql \
    #                  -I$(top_srcdir)/sql \
    #                  -I$(top_builddir)/include \
    #                  -I$(top_builddir)/include/mysql \
    #                  -I$(top_builddir)/sql \
    #                                 $$file 2>/dev/null | \
    #                  /bin/sed -e '/^# /d' \
    #                            -e '/^[    ]*$$/d' \
    #                            -e '/^#pragma GCC set_debug_pwd/d' \
    #                            -e '/^#ident/d' > \
    #                                       $(top_builddir)/abi_check.out; \
    #                  /usr/bin/diff -w $$file.pp $(top_builddir)/abi_check.out; \
    #                  /bin/rm $(top_builddir)/abi_check.out; \
    #done
```

### 二. 添加新的SQL命令（以SHOW DISK_USAGE为例）

#### 1、修改lex.h添加相关Token

```c
static SYMBOL symbols[] = {
  ...
  { "DISK",     SYM(DISK_SYM)},
  /*BEGIN GUOSONG MODIFICATION*/
  /* Reason for Modification: */
  /* Tokens for the SHOW DISK_USAGE command*/
  {"DISK_USAGE", SYM(DISK_USAGE_SYM)},
  /*END GUOSONG MODIFICATION*/
  { "DISTINCT",     SYM(DISTINCT)},
  ...
```

#### 2、修改sql_lex.h添加相关新命令

```c
enum enum_sql_command {
...
  SQLCOM_SHOW_TRIGGERS,
  /*BEGIN GUOSONG MODIFICATION: */
  /*Reason for this modification: */
  /*Add SQLCOM_SHOW_DISK_USAGE reference*/
  SQLCOM_SHOW_DISK_USAGE,
  /*END GUOSONG MODIFICATION*/
...
```

#### 3.修改sql_yacc.yy添加相关token

**添加token**

```c
%token  DISK_SYM
/*BEGIN GUOSONG MODIFICATION*/
/*Reasion for this modification:/*
/*Add DISK_USAGE_SYM */
%token DISK_USAGE_SYM
/*END GUOSONG MODIFICATION*/
%token  DISTINCT                      /* SQL-2003-R */
```

**解析SHOW DISK_USAGE命令**

```c
show:
          SHOW
          {
            LEX *lex=Lex;
            lex->wild=0;
            lex->lock_option= TL_READ;
            mysql_init_select(lex);
            lex->current_select->parsing_place= SELECT_LIST;
            bzero((char*) &lex->create_info,sizeof(lex->create_info));
          }
          show_param
          {}
        ;

show_param:
           /*BEGIN GUOSONG MODIFICATION*/
           /*Reason for this modification:*/
           /*Add show disk usage symbol parsing*/
           DISK_USAGE_SYM
           {
             LEX *lex = Lex;
             lex->sql_command = SQLCOM_SHOW_DISK_USAGE;
           }
            | DATABASES wild_and_where 
           /*END GUOSONG MODIFICATION*/
```

>
注意是在show databases 前面添加，故在DATABASES前面需要添加 | 这个竖线


#### 4.修改sql\_parser.cc添加SHOW DISK_USAGE命令的case

```c
  case SQLCOM_SHOW_AUTHORS:
    res= mysqld_show_authors(thd);
    break;
  /*BEGIN GUOSONG MODIFICATION*/
  /*Reason for this modification*/
  /*Add SQLCOM_SHOW_DISK_USAGE case statement*/
  case SQLCOM_SHOW_DISK_USAGE:
  {
    res = mysqld_show_disk_usage(thd);
    break;
  }
  /*END GUOSONG MODIFICATION*/
```

添加函数声明，目前修改sql_priv.h文件添加

```c
/*BEGIN GUOSONG MODIFICATION*/
/*Reason for Modification*/
/*Declared mysqld_show_disk_usage function*/
bool mysqld_show_disk_usage(THD *thd);
/*END GUOSONG MODIFICATION*/
```



#### 5.在sql\_show.cc中添加mysqld\_show\_disk\_usage

```c
/*BEGIN GUOSONG MODIFICATION*/
/*Reason for Modification*/
/*Add show disk uage method*/
/***************************************************************************
**List all database disk usage
***************************************************************************/

bool mysqld_show_disk_usage(THD *thd)
{
    List<Item> field_list;
    Protocol *protocol = thd->protocol;
    DBUG_ENTER("mysqld_show_disk_usage");
    
    field_list.push_back(new Item_empty_string("Database",50));
    field_list.push_back(new Item_empty_string("Size_in_bytes", 30));

    if(protocol->send_fields(&field_list,
    ¦   ¦   ¦   Protocol::SEND_NUM_ROWS | Protocol::SEND_EOF))
    ¦   DBUG_RETURN(TRUE);

    /*send test data*/
    protocol->prepare_for_resend();
    protocol->store("test_row", system_charset_info);
    protocol->store("1024", system_charset_info);
    if(protocol->write())
    ¦ DBUG_RETURN(TRUE);

    my_eof(thd);
    DBUG_RETURN(FALSE);
}
/*END GUSONG MODIFICATION*/
```

#### 6. 通过bison以及gen\_lex_hash生成相关语法


```shell
bison -y -p MYSQL -d sql_yacc.yy 
[guosong@dev-00 11:18:52 sql]$ll y.tab.
y.tab.c  y.tab.h 
```

使用y.tab.c和y.tab.h替换sql_yacc.cc和sql_yacc.h

```shell
mv y.tab.c sql_yacc.cc 
mv y.tab.h sql_yacc.h
```

通过gen\_lex_hash命令生成新的lex\_hash.h文件

```shell
#在源码的sql目录下
./gen_lex_hash > lex_hash.h
```

#### 7. make install遇到错误

```shell
../include/my_global.h:482:53: error: size of array ‘compile_time_assert’ is negative
                              __attribute__ ((unused));  
```

修改mysqld.cc文件

```c
  /*BEGIN GUOSONG MODIFICATION*/
  /*Reason for Modification*/
  /*Solve make error*/
  compile_time_assert(sizeof(com_status_vars)/sizeof(com_status_vars[0]) - 1 == 
    ¦   ¦   ¦   ¦   ¦SQLCOM_END + 8-1);
  /*END GUOSONG MODIFICATION*/
```

#### 8.测试

```sql
mysql> \s
--------------
mysql  Ver 14.14 Distrib 5.5.42, for Linux (x86_64) using readline 5.1

Connection id:          1
Current database:
Current user:           root@127.0.0.1
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         5.1.45-debug-Guosong Modification Source distribution
Protocol version:       10
Connection:             127.0.0.1 via TCP/IP
Server characterset:    utf8
Db     characterset:    utf8
Client characterset:    utf8
Conn.  characterset:    utf8
TCP port:               5145
Uptime:                 14 sec

Threads: 1  Questions: 5  Slow queries: 0  Opens: 15  Flush tables: 1  Open tables: 8  Queries per second avg: 0.357
--------------

mysql> show DISK_USAGE;
+----------+---------------+
| Database | Size_in_bytes |
+----------+---------------+
| test_row | 1024          |
+----------+---------------+
1 row in set (0.00 sec)
```
