## DDL数据定义

### 创建数据库

```shell
CREATE DATABASE [IF NOT EXISTS] database_name
[COMMENT database_comment]
[LOCATION hdfs_path]
[WITH DBPROPERTIES (property_name=property_value, ...)];
```

```shell
# 1. 数据库在 HDFS 上的默认存储路径是/user/hive/warehouse/*.db
create database db_hive;
# 2. 增加 if not exists 判断
create database if not exists db_hive;
# 3. 指定数据库在 HDFS 上存放的位置
create database db_hive2 location '/db_hive2.db';
```

### 查询数据库

```shell
# 1. 显示数据库
show databases;
show databases like 'db_hive*';
# 2. 查看数据库详情
desc database db_hive;
desc database extended db_hive;
# 3. 切换当前数据库
use db_hive;
```

### 修改数据库

```shell
# 1. 修改数据库
alter database db_hive set dbproperties('createtime'='20170830');
```

### 删除数据库

```shell
# 1. 删除空数据库
drop database db_hive2;
# 2. if exists 判断数据库是否存在
drop database if exists db_hive2;
# 3. 如果数据库不为空，可以采用 cascade 命令，强制删除
drop database db_hive cascade;
```

### 创建表

```shell
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name
[(col_name data_type [COMMENT col_comment], ...)]
[COMMENT table_comment]
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]
[CLUSTERED BY (col_name, col_name, ...)
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS]
[ROW FORMAT row_format]
[STORED AS file_format]
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)]
[AS select_statement]
```

* `CREATE TABLE`创建一个指定名字的表。如果相同名字的表已经存在，则用户可以用`IF NOT EXISTS`选项来忽略这个异常。
* `EXTERNAL`关键字可以让用户创建一个外部表，在建表的同时可以指定一个指向实际数据的路径`LOCATION`，在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。
* `COMMENT`为表和列添加注释。
* `PARTITIONED BY`创建分区表
* `CLUSTERED BY`创建分桶表
* `SORTED BY`不常用，对桶中的一个或多个列另外排序 
* `ROW FORMAT`格式： `DELIMITED [FIELDS TERMINATED BY char] [COLLECTION ITEMS TERMINATED BY char] [MAP KEYS TERMINATED BY char] [LINES TERMINATED BY char] | SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]`
    
    用户在建表的时候可以自定义`SerDe`或者使用自带的`SerDe`。 如果没有指定`ROW FORMAT`或者`ROW FORMAT DELIMITED`，将会使用自带的`SerDe`。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的`SerDe`，Hive通过`SerDe`确定表的具体的列的数据。 
    SerDe是Serialize/Deserilize的简称， hive 使用Serde进行行对象的序列与反序列化。
* `STORED AS`指定存储文件类型 
  
    常用的存储文件类型：`SEQUENCEFILE`（二进制序列文件）、`TEXTFILE`（文本）、`RCFILE`（列式存储格式文件） 
    如果文件数据是纯文本，可以使用`STORED AS TEXTFILE`。如果数据需要压缩，使用`STORED AS SEQUENCEFILE`。
* `LOCATION`：指定表在HDFS上的存储位置。 
* `AS`：后跟查询语句，根据查询结果创建表。
* `LIKE`允许用户复制现有的表结构，但是不复制数据。

#### 管理表

默认创建的表都是所谓的管理表，有时也被称为内部表。因为这种表，Hive会控制着数据的生命周期。 Hive默认情况下会将这些表的数据存储在由配置项`hive.metastore.warehouse.dir`(例如，`/user/hive/warehouse`)所定义的目录的子目录下。
当我们删除一个管理表时，Hive也会删除这个表中数据。管理表不适合和其他工具共享数据。

#### 外部表

因为表是外部表，所以Hive并非认为其完全拥有这份数据。删除该表并不会删除掉这份数据，不过描述表的元数据信息会被删除掉。

通常中间表、结果表使用内部表存储，数据通过`SELECT+INSERT`进入内部表。

```shell
create external table if not exists dept(
  deptno int,
  dname string,
  loc int
)
row format delimited fields terminated by '\t';
create external table if not exists emp(
  empno int,
  ename string,
  job string,
  mgr int,
  hiredate string,
  sal double,
  comm double,
  deptno int
)
row format delimited fields terminated by '\t';
show tables;
desc formatted dept;
drop table dept;
```

#### 管理表与外部表的互相转换

```shell
alter table student2 set tblproperties('EXTERNAL'='TRUE');
alter table student2 set tblproperties('EXTERNAL'='FALSE');
```

注意：`('EXTERNAL'='TRUE')`和`('EXTERNAL'='FALSE')`为固定写法，区分大小写！

### 修改表

```shell
# 1. 重命名表
ALTER TABLE table_name RENAME TO new_table_name
alter table dept_partition2 rename to dept_partition3;
# 2. 增加、修改和删除表分区
create table dept_partition(deptno int, dname string, loc string)
  partitioned by (day string)
  row format delimited fields terminated by '\t';
load data local inpath '/opt/module/hive/datas/dept_20200401.log' into table dept_partition partition(day='20200401');
alter table dept_partition add partition(day='20200404');
alter table dept_partition add partition(day='20200404'),partition(day='20200406');
alter table dept_partition drop partition(day='20200406');
alter table dept_partition drop partition(day='20200404'), partition(day='20200405');
show partitions dept_partition;
desc formatted dept_partition;
# 3. 增加/修改/替换列信息
ALTER TABLE table_name CHANGE [COLUMN] col_old_name col_new_name column_type [COMMENT col_comment] [FIRST|AFTER column_name]
ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type [COMMENT col_comment], ...)
alter table dept add columns(deptdesc string);
alter table dept change column deptdesc desc string;
alter table dept replace columns(deptno string, dname string, loc string);
# 4. 删除表
drop table dept;
```

注：`ADD`是代表新增一字段，字段位置在所有列后面(`partition`列前)， `REPLACE`则是表示替换表中所有字段。

```shell
desc dept;
alter table dept add columns(deptdesc string);
alter table dept change column deptdesc desc string;
alter table dept replace columns(deptno string, dname string, loc string);
```

