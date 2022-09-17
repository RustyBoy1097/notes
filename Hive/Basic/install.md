## 安装 Hive

```shell
# 1. 解压apache-hive-3.1.2-bin.tar.gz到/opt/module/目录下面
tar -zxvf /opt/software/apache-hive-3.1.2-bin.tar.gz -C /opt/module/
mv /opt/software/apache-hive-3.1.2-bin /opt/module/hive
# 2. 修改/etc/profile.d/my_env.sh，添加环境变量
# #HIVE_HOME
# export HIVE_HOME=/opt/module/hive
# export PATH=$PATH:$HIVE_HOME/bin
sudo vim /etc/profile.d/my_env.sh
# 3. 初始化元数据库
bin/schematool -dbType derby -initSchem
# 4. 启动Hive
bin/hive
```
NOTE: Hive默认使用的元数据库为derby，开启Hive之后就会占用元数据库，且不与其他客户端共享数据，所以我们需要将Hive的元数据地址改为MySQL。

### 使用MYSQL作为Metastore

```shell
# 1. 在$HIVE_HOME/conf目录下新建hive-site.xml文件
vim $HIVE_HOME/conf/hive-site.xml
# 2. 初始化Hive元数据库
schematool -initSchema -dbType mysql -verbose
# 3. 启动hive
bin/hive
```

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <!-- jdbc 连接的 URL -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://hadoop102:3306/metastore?useSSL=false</value>
    </property>
    <!-- jdbc 连接的 Driver-->
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <!-- jdbc 连接的 username-->
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>
    <!-- jdbc 连接的 password -->
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>000000</value>
    </property>
    <!-- Hive 元数据存储版本的验证 -->
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
    <!--元数据存储授权-->
    <property>
        <name>hive.metastore.event.db.notification.api.auth</name>
        <value>false</value>
    </property>
    <!-- Hive 默认在 HDFS 的工作目录 -->
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>
</configuration>
```
### 使用元数据服务的方式访问Hive


添加配置信息到`hive-site.xml`

```xml
<!-- 指定存储元数据要连接的地址 -->
<property>
    <name>hive.metastore.uris</name>
    <value>thrift://hadoop102:9083</value>
</property>
```
启动 metastore

```shell
hive --service metastore
```

### 使用JDBC方式访问Hive

添加配置信息到`hive-site.xml`

```
<!-- 指定 hiveserver2 连接的 host -->
<property>
    <name>hive.server2.thrift.bind.host</name>
    <value>hadoop102</value>
</property>
<!-- 指定 hiveserver2 连接的端口号 -->
<property>
    <name>hive.server2.thrift.port</name>
    <value>10000</value>
</property>
```

```shell
# 1. 启动 hiveserver2
bin/hive --service hiveserver2
# 2. 启动 beeline 客户端
bin/beeline -u jdbc:hive2://hadoop102:10000 -n 12345
```

### hive 服务启动脚本

```shell
#!/bin/bash

# hiveservices.sh
HIVE_LOG_DIR=$HIVE_HOME/logs
if [ ! -d $HIVE_LOG_DIR ]
then
  mkdir -p $HIVE_LOG_DIR
fi
#检查进程是否运行正常，参数 1 为进程名，参数 2 为进程端口
function check_process() {
  pid=$(ps -ef 2>/dev/null | grep -v grep | grep -i $1 | awk '{print $2}')
  ppid=$(netstat -nltp 2>/dev/null | grep $2 | awk '{print $7}' | cut -d '/' -f 1)
  echo $pid
  [[ "$pid" =~ "$ppid" ]] && [ "$ppid" ] && return 0 || return 1
}

function hive_start() {
  metapid=$(check_process HiveMetastore 9083)
  cmd="nohup hive --service metastore >$HIVE_LOG_DIR/metastore.log 2>&1 &"
  [ -z "$metapid" ] && eval $cmd || echo "Metastroe 服务已启动"
  server2pid=$(check_process HiveServer2 10000)
  cmd="nohup hiveserver2 >$HIVE_LOG_DIR/hiveServer2.log 2>&1 &"
  [ -z "$server2pid" ] && eval $cmd || echo "HiveServer2 服务已启动"
}

function hive_stop() {

  metapid=$(check_process HiveMetastore 9083)
  [ "$metapid" ] && kill $metapid || echo "Metastore 服务未启动"
  server2pid=$(check_process HiveServer2 10000)
  [ "$server2pid" ] && kill $server2pid || echo "HiveServer2 服务未启动"
}

case $1 in
  "start")
    hive_start
    ;;
  "stop")
    hive_stop
    ;;
  "restart")
    hive_stop
    sleep 2
    hive_start
    ;;
  "status")
    check_process HiveMetastore 9083 >/dev/null && echo "Metastore 服务运行正常" || echo "Metastore 服务运行异常"
    check_process HiveServer2 10000 >/dev/null && echo "HiveServer2 服务运行正常" || echo "HiveServer2 服务运行异常"
    ;;
  *)
    echo Invalid Args!
    echo 'Usage: '$(basename $0)' start|stop|restart|status'
    ;;
esac
```

```shell
chmod +x $HIVE_HOME/bin/hiveservices.sh
hiveservices.sh start
```





