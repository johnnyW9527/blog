#### 11 redis集群（cluster）

##### 11.1 存在问题

```
单台redis容量限制，如何进行扩容？继续加内存、加硬件么?
单台redis并发写量太大有性能瓶颈，如何解决？
redis3.0中提供了集群可以解决这些问题.
```

##### 11.2 what is cluster

> redis集群是对redis的水平扩容，即启动N个redis节点，将整个数据分布存储在这个N个节点中，每个节点存储总数据的1/N。
>
> 由3台master和3台slave组成的redis集群，每台master承接客户端三分之一请求和写入的数据，当master挂掉后，slave会自动替代master，做到高可用。

![](C:\study\mlog\picture\6.png)

##### 11.3 How to do

###### 11.3.1 配置3主3从集群

###### 11.3.2 创建案例工作目录: cluster

> 执行下面命令创建 /opt/cluster 目录，本次所有操作，均在 cluster 目录进行

```
# 方便演示，停止所有的redis
ps -ef | grep redis | awk -F" " '{print $2;}' | xargs kill -9
mkdir /opt/cluster
cd /opt/cluster/
```

###### 11.3.3 将redis.conf复制到cluster目录

```
cp /opt/redis-6.2.1/redis.conf /opt/cluster/
```

###### 11.3.4 创建master1的配置文件：redis-6379.conf

```
include /opt/cluster/redis.conf
daemonize yes
bind 192.168.200.129
dir /opt/cluster/
port 6379
dbfilename dump_6379.rdb
pidfile /var/run/redis_6379.pid
logfile "./6379.log"
# 开启集群设置
cluster-enabled yes
# 设置节点配置文件
cluster-config-file node-6379.conf
# 设置节点失联时间，超过该时间(毫秒)，集群自动进行主从切换
cluster-node-timeout 15000
```

###### 11.3.5 创建master2的配置 文件：redis-6380.conf

```
include /opt/cluster/redis.conf
daemonize yes
bind 192.168.200.129
dir /opt/cluster/
port 6380
dbfilename dump_6380.rdb
pidfile /var/run/redis_6380.pid
logfile "./6380.log"
# 开启集群设置
cluster-enabled yes
# 设置节点配置文件
cluster-config-file node-6380.conf
# 设置节点失联时间，超过该时间(毫秒)，集群自动进行主从切换
cluster-node-timeout 15000
```

###### 11.3.6 创建master3的配置文件：redis-6381.conf

```
include /opt/cluster/redis.conf
daemonize yes
bind 192.168.200.129
dir /opt/cluster/
port 6381
dbfilename dump_6381.rdb
pidfile /var/run/redis_6381.pid
logfile "./6381.log"
# 开启集群设置
cluster-enabled yes
# 设置节点配置文件
cluster-config-file node-6381.conf
# 设置节点失联时间，超过该时间(毫秒)，集群自动进行主从切换
cluster-node-timeout 15000
```

###### 11.3.7 创建slave1的配置文件：redis-6389.conf

```
include /opt/cluster/redis.conf
daemonize yes
bind 192.168.200.129
dir /opt/cluster/
port 6389
dbfilename dump_6389.rdb
pidfile /var/run/redis_6389.pid
logfile "./6389.log"
# 开启集群设置
cluster-enabled yes
# 设置节点配置文件
cluster-config-file node-6389.conf
# 设置节点失联时间，超过该时间(毫秒)，集群自动进行主从切换
cluster-node-timeout 15000
```

###### 11.3.8 创建slave2的配置文件：redis-6390.conf

```
include /opt/cluster/redis.conf
daemonize yes
bind 192.168.200.129
dir /opt/cluster/
port 6390
dbfilename dump_6390.rdb
pidfile /var/run/redis_6390.pid
logfile "./6390.log"
# 开启集群设置
cluster-enabled yes
# 设置节点配置文件
cluster-config-file node-6390.conf
# 设置节点失联时间，超过该时间(毫秒)，集群自动进行主从切换
cluster-node-timeout 15000
```

###### 11.3.9 创建slave3的配置文件：redis-6391.conf

```
include /opt/cluster/redis.conf
daemonize yes
bind 192.168.200.129
dir /opt/cluster/
port 6391
dbfilename dump_6391.rdb
pidfile /var/run/redis_6391.pid
logfile "./6391.log"
# 开启集群设置
cluster-enabled yes
# 设置节点配置文件
cluster-config-file node-6391.conf
# 设置节点失联时间，超过该时间(毫秒)，集群自动进行主从切换
cluster-node-timeout 15000
```

###### 11.3.10 确保node-xxxx.conf文件已正常生成

```
稍后我们会将6个实例合并到一个集群，在组合之前，我们要确保6个redis实例启动后，nodesxxxx.conf文件都生成正常，如下， /opt/cluster 目录中确实都生成成功了
```

###### 11.3.11 将6个节点合成一个集群

执行下面命令，将6个redis合体

```
/opt/redis-6.2.1/src/redis-cli --cluster create --cluster-replicas 1
192.168.200.129:6379 192.168.200.129:6380 192.168.200.129:6381
192.168.200.129:6389 192.168.200.129:6390 192.168.200.129:6391
```

> * 合体的命令后面会跟上所有节点的ip:port列表，多个之间用空格隔开，注意ip不要写127.0.0.1，要写真实ip
> * --cluster-replicas 1：表示采用最简单的方式配置集群，即每个master配1个slave，6个节点就形成了3主3从

###### 11.3.12 连接集群节点，查看集群信息： cluster nodes

> 需要使用 redis-cli -c 命令连接集群中6个节点中任何一个节点都可以，注意和之前的连接参数有点不同 redis-cli 命令后面多了一个 -c 参数，表示采用集群的方式连接，连上以后，然后使用 cluster nodes 可以查看集群节点信息，如下

##### 11.4 redis集群如何分配这6个节点？

> 一个集群至少有3个主节点，因为新master的选举需要大于半数的集群master节点同意才能选举成功，如果只有两个master节点，当其中一个挂了，是达不到选举新master的条件的。选项--cluster-replicas 1表示我们希望为集群中的每个主节点创建一个从节点。分配原则尽量保证每个主库运行在不同的ip，每个主库和从库不在一个ip上，这样才能做到高可用。

##### 11.5 什么是slots(槽)

> Redis集群内部划分了16384个slots（插槽），合并的时候，会将每个slots映射到一个master上面，比如上面3个master和slots的关系如下

| redis主节点          | 槽位范围                                      |
| -------------------- | --------------------------------------------- |
| master1(端口：6379)  | [0-5460]，插槽的位置从0开始的，0表示第1个插槽 |
| master2(端口：6380)  | [5460-10922]                                  |
| master3(端口：6381)  | [10923-16383]                                 |
| slave1,slave2,slave3 | 从节点没有槽位，slave是用来对master做替补的   |

> 而数据库中的每个key都属于16384个slots中的其中1个，当通过key读写数据的时候，redis需要先根据key计算出key对应的slots，然后根据slots和master的映射关系找到对应的redis节点，key对应的数据就在这个节点上面
>
> 集群中使用公式 CRC16(key)%16384 计算key属于哪个槽

##### 11.6 在集群中录入数据

>在 redis-cli 每次录入、查询键值，redis都会计算key对应的插槽，如果不是当前redis节点的插槽，redis会报错，并告知应前往的redis实例地址和端口，效果如下，我们连接了6379这个实例来操作k1，这个节点发现k1的槽位在6381上面，返回了错误信息，怎么办呢?

```
[root@hspEdu01 cluster]# redis-cli -h 192.168.200.129 -p 6379
192.168.200.129:6379> set k1 v1
(error) MOVED 12706 192.168.200.129:6381
```

> 使用redis-cli客户端提供了-c参数可以解决这个问题，表示以集群方式执行，执行命令的时候当前节点处理不了的时候，会自动将请求重定向到目标节点，效果如下，被重定向到6381了

```
[root@hspEdu01 cluster]# redis-cli -c -h 192.168.200.129 -p 6379
192.168.200.129:6379> set k1 v1
-> Redirected to slot [12706] located at 192.168.200.129:6381
OK
192.168.200.129:6381>
[root@hspEdu01 cluster]# redis-cli -c -h 192.168.200.129 -p 6379
192.168.200.129:6379> get k1
-> Redirected to slot [12706] located at 192.168.200.129:6381
"v1"
192.168.200.129:6381>

```

> 不在一个slot下面，不能使用mget、mset等多键操作，效果如下

```
192.168.200.129:6381> mset k1 v1 k2 v2
(error) CROSSSLOT Keys in request don't hash to the same slot
192.168.200.129:6381> mget k1 k2
(error) CROSSSLOT Keys in request don't hash to the same slot
```

> 可以通过{}来定义组的概念，从而使key中{}内相同的键值放到一个slot中去，效果如下

```
192.168.200.129:6381> mset k1{g1} v1 k2{g1} v2 k3{g1} v3
OK
192.168.200.129:6381> mget k1{g1} k2{g1} k3{g1}
1) "v1"
2) "v2"
3) "v3"
```

##### 11.7 slot相关的一些命令

* cluster keyslot <key>：计算key对应的slot
* cluster coutkeysinslot <slot>：获取slot槽位中key的个数
* cluster getkeysinslot <slot> <count> 返回count个slot槽中的键

```
192.168.200.129:6381> cluster keyslot k1{g1}
(integer) 13519
192.168.200.129:6381> cluster countkeysinslot 13519
(integer) 3
192.168.200.129:6381> cluster getkeysinslot 13519 3
1) "k1{g1}"
2) "k2{g1}"
3) "k3{g1}"
```

##### 11.8 故障恢复

> 如果主节点下线，从节点是否能够提升为主节点？注意：要等15秒
>
> 下面我们来试试，如下，连接master1，然后将master1停掉

```
[root@hspEdu01 cluster]# redis-cli -c -h 192.168.200.129 -p 6379
192.168.200.129:6379> shutdown
not connected>
```

> 执行下面命令，连接master1，看下集群节点的信息

```
redis-cli -c -h 192.168.200.129 -p 6380
cluster nodes
```

```
[root@localhost cluster]# /usr/local/bin/redis-cli -c -h 192.168.203.100 -p 6380
192.168.203.100:6380> CLUSTER NODES
36f60cd7a93e9661504467a726e9337259f15468 192.168.203.100:6379@16379 master - 0 1703774112000 1 connected 0-5460
f1455311cea63ff675459e524a9a5df946d8fa8a 192.168.203.100:6391@16391 master - 0 1703774113044 7 connected 10923-16383
b932689d2159a4c77949c4c56286e16154f2fa27 192.168.203.100:6380@16380 myself,master - 0 1703774111000 2 connected 5461-10922
d53aad79b0153ba7f0376377271ad4a2a866a34f 192.168.203.100:6389@16389 slave 36f60cd7a93e9661504467a726e9337259f15468 0 1703774110947 1 connected
04ee37efbb3f66559c412bfb1de4c868785e7883 192.168.203.100:6390@16390 slave b932689d2159a4c77949c4c56286e16154f2fa27 0 1703774109922 2 connected
8ab408be63926dafeab495c949be000ebd56e937 192.168.203.100:6381@16381 master,fail - 1703774052850 1703774049623 3 disconnected

```

**我们再启动6381，6381变成slave挂在6391下面**

 ```
[root@localhost cluster]# /usr/local/bin/redis-server ./redis-6381.conf
[root@localhost cluster]# /usr/local/bin/redis-cli -c -h 192.168.203.100 -p 6380
192.168.203.100:6380> CLUSTER NODES
36f60cd7a93e9661504467a726e9337259f15468 192.168.203.100:6379@16379 master - 0 1703774198000 1 connected 0-5460
f1455311cea63ff675459e524a9a5df946d8fa8a 192.168.203.100:6391@16391 master - 0 1703774199000 7 connected 10923-16383
b932689d2159a4c77949c4c56286e16154f2fa27 192.168.203.100:6380@16380 myself,master - 0 1703774198000 2 connected 5461-10922
d53aad79b0153ba7f0376377271ad4a2a866a34f 192.168.203.100:6389@16389 slave 36f60cd7a93e9661504467a726e9337259f15468 0 1703774196861 1 connected
04ee37efbb3f66559c412bfb1de4c868785e7883 192.168.203.100:6390@16390 slave b932689d2159a4c77949c4c56286e16154f2fa27 0 1703774200012 2 connected
8ab408be63926dafeab495c949be000ebd56e937 192.168.203.100:6381@16381 slave f1455311cea63ff675459e524a9a5df946d8fa8a 0 1703774198961 7 connected

 ```

> **如果某一段插槽的主从都宕机了，**redis**服务是否还能继续**
>
> 这个时候要看 cluster-require-full-coverage 参数的值了
>
> * yes（默认值） :整个集群都无法提供服务了
> * no: 宕机的这部分槽位数据全部不能使用，其他槽位正常



