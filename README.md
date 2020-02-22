# redis-cluster-docker

本项目主要目的是利用docker-compose来在单机上，一键搭建redis集群。规模为“一主二从三集群”。

首先先把基础概念内容相对理一遍：[relevant](./relevant.md)



**前提：本试验环境已经提前安装了docker和docker-compose**

**说明：本次部署是单机伪集群，想要部署真正的集群，修改ip地址拆分配置到各个主机上部署即可。**

## 部署



这边先说一些部署技巧:

> 部署至少3个且奇数个的Sentinel节点：
>
> 1. 3个以上是通过Sentinel节点的个数提高对于故障断定的准确性，因为领导者选举需要至少一般加1个的节点，奇数个节点可以在满足该条件的基础上节省一个节点。详情见前面提及的概念内容。
> 2. Sentinel节点集合可以只监控一个主节点，也可以监控多个主节点。

项目分成两个目录，一个是redis目录。主要存放redis主从节点的docke-compose文件。

1. 在redis文件夹下创建redis目录，主要是为了持久化redis。（RDB和AOF路径一样）

2. 在同文件夹下新建 docker-compose.yml文件。

   ```shell
   ├── data
   │   ├── master
   │   ├── slave1
   │   └── slave2
   └── docker-compose.yml
   ```

3. 这边简单说重要的配置项：

   ```yaml
   ...
       container_name: redis-master
   ...
       volumes:
         - ./data/master:/data
       command: redis-server --port 6379 --requirepass 123
       
    ...
       container_name: redis-slave-1
   ...
       volumes:
         - ./data/slave1:/data
       command: redis-server --slaveof 192.168.0.107 6379 --port 6380 --requirepass 1234 --masterauth 123
       
   ```

   > 需要配置好对应验证密码和容器命以便从节点连接主节点。注意从节点连接不能用127.0.0.1连接上主服务器。这个的原因就是redis主服务器绑定了127.0.0.1，那么跨服务器IP的访问就会失败，从服务器用IP和端口访问主的时候，主服务器发现本机6379端口绑在了127.0.0.1上，也就是只能本机才能访问，外部请求会被过滤。这边我修改为ifconfig获取的网卡地址，如果是线上生产环境建议绑定IP地址。

接下来是在跟目录下建立文件夹sentinel：

1. 在文件夹下建立配置文件夹conf。对于3个配置文件：slave1.conf，slave2.conf，slave3.conf内容相同：

```
sentinel monitor mymaster 192.168.0.107 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel auth-pass mymaster 1234
sentinel failover-timeout mymaster 180000
sentinel deny-scripts-reconfig yes
```

2. 在文件夹下新建docker-compose.yml。现在sentinel文件夹结构如下：

   ```shell
   ├── conf
   │   ├── sentinel1.conf
   │   ├── sentinel2.conf
   │   └── sentinel3.conf
   └── docker-compose.yml
   ```

3. 配置项重点是以下2条（注意配置文件每次成功运行后会自动更新一些节点信息）：

   ```
   #自定义集群名，其中 192.168.8.188 为 redis-master 的 ip，6379 为 redis-master 的端口，2 为最小投票数（因为有 3 台 Sentinel 所以可以设置成 2）
   sentinel monitor mymaster 192.168.8.188 6379 2
   # 每个Sentinel节点都要通过定期发送ping命令来判断redis数据节点和其余Sentinel节点是否可达，超过down-after-milliseconds即为不可达。
   sentinel down-after-milliseconds mymaster 30000
   ```

   