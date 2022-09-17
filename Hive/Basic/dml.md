## 数据导入

```shell
# 1. 向表中装载数据（Load）
load data [local] inpath '数据的 path' [overwrite] into table student [partition (partcol1=val1,…)];
# 2. 通过查询语句向表中插入数据（Insert）
insert into table student_par values(1,'wangwu'),(2,'zhaoliu');
insert overwrite table student_par select id, name from student where month='201709';
from student insert overwrite table student partition(month='201707') select id, name where month='201709'
# 3. 查询语句中创建表并加载数据（As Select）
create table if not exists student3 as select id, name from student;
# 4. 创建表时通过 Location 指定加载数据路径
create external table if not exists student5( id int, name string)
row format delimited fields terminated by '\t'
location '/student'; # hdfs数据路径
# 5. Import 数据到指定 Hive 表中 
import table student2 from '/user/hive/warehouse/export/student'; # 注意：先用 export 导出后，再将数据导入。 
```

## 数据导出

```shell
# 1. Insert 导出
insert overwrite local directory '/opt/module/hive/data/export/student' select * from student;
insert overwrite local directory '/opt/module/hive/data/export/student1' ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' select * from student;
insert overwrite directory '/user/student2' ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' select * from student;
# 2. Hadoop 命令导出到本地
dfs -get /user/hive/student/student.txt /opt/module/data/export/student3.txt;
# 3. Hive Shell 命令导出
bin/hive -e 'select * from default.student;' > /opt/module/hive/data/export/student4.txt;
# 4. Export 导出到 HDFS 上
export table default.student to '/user/hive/warehouse/export/student';
# 5. Sqoop等工具导出
# 6. 清除表中数据（Truncate）
truncate table student; # 注意：Truncate 只能删除管理表，不能删除外部表中数据
```