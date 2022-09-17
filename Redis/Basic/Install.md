## Linux

### FROM SOURCE

```bash
wget http://download.redis.io/releases/redis-6.0.8.tar.gz
tar xzf redis-6.0.8.tar.gz
cd redis-6.0.8
make
```

### Ubuntu

```
sudo apt update && sudo apt install redis-server
```

### Mac

```
brew install redis
```

### Docker

```
docker pull redis
mkdir -p /mydata/redis/conf
touch /mydata/redis/conf/redis.conf
echo "appendonly yes" >> /mydata/redis/conf/redis.conf
docker run -p 6379:6379 --name redis -v /mydata/redis/data:/data -v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf --restart=always --network common-network -d redis redis-server /etc/redis/redis.conf
// 进入redis容器内部 进行查看或者操作
docker exec -it redis /bin/bash
// 连接redis
docker exec -it redis redis-cli
// 重启redis
docker restart redis
// 设置redis跟随docker启动
docker update redis --restart=always
```