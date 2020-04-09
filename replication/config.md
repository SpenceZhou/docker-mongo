# mongodb 复制集的配置


复制集rs的情况： primary + secondary + arbiter  (主+从+选举节点：参照以下图片)

![avatar](https://docs.mongodb.com/manual/_images/replica-set-primary-with-secondary-and-arbiter.bakedsvg.svg)



## 1. 生成keyfile 用于复制集内部的权限控制
```
openssl rand -base64 741 > mongodb-keyfile

chmod 600 mongodb-keyfile

复制 mongodb-keyfile 文件后再分别执行chmod 操作
chown 999 mongodb-keyfile
```

## 2. 运行docker-compose
```
docker-compose up -d
```
## 3. rs配置
```
docker exec -it mongo-rs1-node1-test /bin/bash

mongo
use admin
db.auth('root','mc-MG-2020-PWD-test')

rs.initiate(
  {
    _id : "rs1",
    members: [
      { _id : 0, host : "192.168.1.247:22217", priority: 10},
      { _id : 1, host : "192.168.1.247:22218", priority: 1}
    ]
  }
)

说明 priority 默认值为1， 数值越大约有可能被选举为主节点

### 添加选举节点 (如果不是主节点，请在主节点运行)(可选)
rs.addArb("192.168.1.247:22219")（暂时不用）

### 其他增加 删除节点的命令

rs.add("192.168.1.247:22218")
rs.remove("192.168.1.247:22218")
```

## 4. stop
```
docker stop mongo-rs1-node1-test
docker stop mongo-rs1-node2-test
```

## 5. remove
```
docker rm mongo-rs1-node1-test
docker rm mongo-rs1-node2-test
```
