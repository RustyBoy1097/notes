## 1. 原生Zk Java API

添加如下Jar依赖

```
// zookeeper-3.4.6.jar
// zookeeper-3.4.6-javadoc.jar
// zookeeper-3.4.6-sources.jar
```

maven

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.6</version>
</dependency>
```

## 开源客户端

```xml
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.2</version>
</dependency>
```

Curator客户端

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>2.4.2</version>
</dependency>
```