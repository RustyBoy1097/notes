## Hive常用交互命令

```shell
# 1. 帮助
bin/hive -help

usage: hive
  -d,--define <key=value> Variable subsitution to apply to hive commands. e.g. -d A=B or --define A=B
  --database <databasename> Specify the database to use
  -e <quoted-query-string> SQL from command line
  -f <filename> SQL from files
  -H,--help Print help information
  --hiveconf <property=value> Use value for given property
  --hivevar <key=value> Variable subsitution to apply to hive commands. e.g. --hivevar A=B
  -i <filename> Initialization SQL file
  -S,--silent Silent mode in interactive shell
  -v,--verbose  Verbose mode (echo executed SQL to the console)
# 2. 退出 hive 窗口
exit;
quit;
# 3. dfs查看hdfs文件系统
dfs  -ls /;
```

## 常见属性配置

```shell
# 修改日志配置信息/opt/module/hive/conf/hive-log4j2.properties.template
mv hive-log4j2.properties.template hive-log4j2.properties
```

```
<property>
    <name>hive.cli.print.header</name>
    <value>true</value>
</property>

<property>
    <name>hive.cli.print.current.db</name>
    <value>true</value>
</property>
```
## 参数配置方式

1. 配置文件方式

    * 默认配置文件：hive-default.xml 
    * 用户自定义配置文件：hive-site.xml
    * Hadoop的配置
2. 命令行参数方式
   启动 Hive 时，可以在命令行添加`-hiveconf param=value` 来设定参数。
3. 参数声明方式
    
    ```shell
    set mapred.reduce.tasks=100;
    # 查看 set mapred.reduce.tasks;
    ```