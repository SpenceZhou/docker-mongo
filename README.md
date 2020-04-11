# docker-mongo

## mongodb 复制集配置

复制集rs的情况： primary + secondary + arbiter  (主+从+选举节点：参照以下图片)

![avatar](https://docs.mongodb.com/manual/_images/replica-set-primary-with-secondary-and-arbiter.bakedsvg.svg)



### 1. 生成keyfile 用于复制集内部的权限控制

参照官方文档  https://docs.mongodb.com/manual/tutorial/deploy-replica-set-with-keyfile-access-control/index.html
```
openssl rand -base64 756 > mongodb-keyfile

chmod 400 mongodb-keyfile


复制 mongodb-keyfile 文件后再分别执行chmod 操作
chown 999 mongodb-keyfile
```

### 2. 运行 mongodb docker
```
docker run -d --name test-rs1 -p 20017:27017 -v `pwd`/data/db:/data/db -v `pwd`/mongodb-keyfile:/opt/mongodb-keyfile -v /etc/localtime:/etc/localtime:ro --privileged=true -e MONGO_INITDB_ROOT_USERNAME=root -e MONGO_INITDB_ROOT_PASSWORD=test-324 mongo:4.2.5 --auth --keyFile /opt/mongodb-keyfile --oplogSize 10240 --replSet rs1

```
### 3. rs配置
```
docker exec -it test-rs1 /bin/bash

mongo
use admin
db.auth('root','test-324')

rs.initiate(
  {
    _id : "rs1",
    members: [
      { _id : 0, host : "192.168.1.xx:20017", priority: 10},
      { _id : 1, host : "192.168.1.xx:22218", priority: 1}
    ]
  }
)

说明 priority 默认值为1， 数值越大约有可能被选举为主节点

### 添加选举节点 (如果不是主节点，请在主节点运行)(可选)
rs.addArb("192.168.1.xx:22219")（暂时不用）

### 其他增加 删除节点的命令

rs.add("192.168.1.xx:22218")
rs.remove("192.168.1.xx:22218")
```

### 4. 配置延时节点（常用于数据备份）

参照官方文档

https://docs.mongodb.com/manual/core/replica-set-delayed-member/index.html

```
rs.add({
   "host" : <hostname:port>,
   "priority" : 0,
   "slaveDelay" : <seconds>,
   "hidden" : true
})

```

### 5. stop&remove
```
docker stop test-rs1
docker rm test-rs1
```

## mongodb 分片配置

参照 sharding/config.md 文件说明
