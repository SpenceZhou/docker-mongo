# 服务器情况

服务器0 192.168.1.230

服务器1 192.168.1.231

服务器2 192.168.1.232

# Mongo 部署情况
服务器0  config-server rs1-data(默认主选举高优先级) rs2-arbiter                rs3-data                    router

服务器1  config-server rs1-data                   rs2-data(默认主选举高优先级) rs3-arbiter                 router

服务器2  config-server rs1-arbiter                rs2-data                   rs3-data (默认主选举高优先级) router


# Mongo集群说明
共有三个 shard  shard1(rs1)  shard2(rs1)  shard3(rs3)

复制集rs的情况： primary + secondary + arbiter  (主+从+选举节点：参照以下图片)

![avatar](https://docs.mongodb.com/manual/_images/replica-set-primary-with-secondary-and-arbiter.bakedsvg.svg)



# 容量计算
集群中包含 9 个节点，其中6个数据节点、3个选举节点

每个复制集 1主1从1选举

每个服务器 2个数据节点 1个选举节点

磁盘有效利用率为： 1/2

服务器 0 可用磁盘 860G

服务器 1 可用磁盘 860G

服务器 2 可用磁盘 860G

故本集群的有效可用磁盘理论值为： 860G *3 / 2 = 1290G =1.25T

# 创建集群

## 1. Mongo根目录
```
cd /home/maicang/docker/mongo
```
## 2. Mongo 权限
```
openssl rand -base64 741 > mongodb-keyfile

chmod 600 mongodb-keyfile
```
#### 复制 mongodb-keyfile 文件后再分别执行chown 操作
```
sudo chown 999 mongodb-keyfile
```
## 3.创建文件夹
```
## 服务器0

mkdir configsrv_db
mkdir rs1_node_db
mkdir rs2_arbiter_db
mkdir rs3_node_db

## 服务器1
mkdir configsrv_db
mkdir rs1_node_db
mkdir rs2_node_db
mkdir rs3_arbiter_db

## 服务器2
mkdir configsrv_db
mkdir rs1_arbiter_db
mkdir rs2_node_db
mkdir rs3_node_db
```

## 4. 运行docker-compose
```
sudo docker-compose up -d
```
## 5. configrs配置
```
sudo docker exec -it mongo-configsrv /bin/bash

mongo
use admin
db.auth('root','mc-MG-2020-PWD')
rs.initiate(
  {
    _id: "configrs",
    configsvr: true,
    members: [
      { _id : 0, host : "192.168.1.230:27018" },
      { _id : 1, host : "192.168.1.231:27018" },
      { _id : 2, host : "192.168.1.232:27018" }
    ]
  }
)
```

## 6. rs1配置  在服务器0运行以下命令
```
sudo docker exec -it mongo-rs1-node /bin/bash

mongo

use admin
db.auth('root','mc-MG-2020-PWD')
rs.initiate(
  {
    _id : "rs1",
    members: [
      { _id : 0, host : "192.168.1.230:27019", priority: 10},
      { _id : 1, host : "192.168.1.231:27019", priority: 1}
    ]
  }
)

说明 priority 默认值为1， 数值越大约有可能被选举为主节点

## 添加选举节点 (如果不是主节点，请在主节点运行)
rs.addArb("192.168.1.232:27019")
## 其他增加 删除节点的命令

rs.add("192.168.1.230:27019")
rs.remove("192.168.1.231:29019")

rs.reconfig(
  {
    _id : "rs1",
    protocolVersion: 1,
    members: [
      { _id : 1, host : "192.168.1.231:27019" },
    ]
  },{"force":true}
)
```
## 8. rs2配置    在服务器1运行以下命令
```
sudo docker exec -it mongo-rs2-node /bin/bash

mongo

use admin
db.auth('root','mc-MG-2020-PWD')
rs.initiate(
  {
    _id : "rs2",
    members: [
      { _id : 0, host : "192.168.1.231:28019", priority: 10 },
      { _id : 1, host : "192.168.1.232:28019", priority: 1 }
    ]
  }
)

## 添加选举节点 (如果不是主节点，请在主节点运行)
rs.addArb("192.168.1.230:28019")
```
## 9. rs3配置    在服务器2运行以下命令
```
sudo docker exec -it mongo-rs3-node /bin/bash

mongo

use admin
db.auth('root','mc-MG-2020-PWD')
rs.initiate(
  {
    _id : "rs3",
    members: [
      { _id : 0, host : "192.168.1.232:29019", priority: 10 },
      { _id : 1, host : "192.168.1.230:29019", priority: 1 }
    ]
  }
)

## 添加选举节点  (如果不是主节点，请在主节点运行)
rs.addArb("192.168.1.231:29019")
```

## 10. 配置router 增加Shard节点
```
sudo docker exec -it mongo-router /bin/bash

mongo

use admin
db.auth('root','mc-MG-2020-PWD')

sh.addShard("rs1/192.168.1.230:27019,192.168.1.231:27019")
sh.addShard("rs2/192.168.1.231:28019,192.168.1.232:28019")

sh.addShard("rs3/192.168.1.232:29019,192.168.1.230:29019")
```

## 11. 测试
```
sh.enableSharding("test")
sh.shardCollection("test.user", { _id : "hashed" } )

use test
for(i=1;i<=10000;i++){db.user.insert({"id":i,"name":"jack"+i})} #模拟往test数据库的user表写入10万数据
```


## 12. 停止容器并销毁数据
```
sudo docker stop mongo-configsrv
sudo docker rm mongo-configsrv

sudo docker stop mongo-rs1-node
sudo docker rm mongo-rs1-node

sudo docker stop mongo-rs2-node
sudo docker rm mongo-rs2-node

sudo docker stop mongo-rs3-node
sudo docker rm mongo-rs3-node


sudo docker stop mongo-rs1-arbiter
sudo docker rm mongo-rs2-arbiter

sudo docker stop mongo-rs2-arbiter
sudo docker rm mongo-rs2-arbiter

sudo docker stop mongo-rs3-arbiter
sudo docker rm mongo-rs3-arbiter

sudo docker stop mongo-router
sudo docker rm  mongo-router

sudo rm -rf configsrv_db
sudo rm -rf router_db
sudo rm -rf rs1_node_db
sudo rm -rf rs2_node_db
sudo rm -rf rs3_node_db
sudo rm -rf rs1_arbiter_db
sudo rm -rf rs2_arbiter_db
sudo rm -rf rs3_arbiter_db
```
