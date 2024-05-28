#### 10 redis主从复制

##### 10.1 what

> 主机更新后根据配置和策略，自动同步到备机的master/slave机制，Master以写为主，Slave以读为主

##### 10.2 todo

* 读写分离，性能扩展，降低主服务器的压力
* 容灾，快速恢复，主机挂掉时，从机变为主机

![](C:\study\mlog\picture\4.png)

##### 10.3 How 

###### 10.3.1 配置1主2从

| 角色       | 端口 |
| ---------- | ---- |
| master(主) | 6379 |
| slave1(从) | 6380 |
| slave2(从) | 6381 |

###### 10.3.2 配置

1. 创建案例工作目录：master-slave

> 执行下面命令创建 /opt/master-slave 目录，本次所有操作，均在 master-slave 目录进行

```
ps -ef | grep redis | awk -F" " '{print $2;}' | xargs kill -9 # 方便演示，停止所有的
redis
mkdir /opt/master-slave
cd /opt/master-slave/
```

2. 将redis.conf复制到master-salve目录

```
cp /opt/redis-6.2.1/redis.conf /opt/master-slave/
```

3. 创建master的配置文件：redis-6379.conf

> 在/opt/master-slave目录创建 redis-6379.conf 文件，内容如下，注意 192.168.203.100这个测试机器的ip

```
#redis.conf是redis原配置文件，内部包含了很多默认的配置，这里使用include将其引用，相当于把
redis.conf内容直接贴进来了
include /opt/master-slave/redis.conf
daemonize yes
bind 192.168.200.129
#配置密码
requirepass 123456
dir /opt/master-slave/
logfile /opt/master-slave/6379.log
#端口
port 6379
#rdb文件
dbfilename dump_6379.rdb
#pid文件
pidfile /var/run/redis_6379.pid
```

4. 创建slave1的配置文件：redis-6380.conf

> 在/opt/master-slave目录创建 redis-6380.conf 文件，内容如下

```
include /opt/master-slave/redis.conf
daemonize yes
bind 192.168.200.129
requirepass 123456
dir /opt/master-slave/
port 6380
dbfilename dump_6380.rdb
pidfile /var/run/redis_6380.pid
logfile /opt/master-slave/6380.log
#用来指定主机：slaveof 主机ip 端口
slaveof 192.168.200.129 6379
#主机的密码
masterauth 123456
```

5. 创建slave2的配置文件：redis-6381.conf

```
include /opt/master-slave/redis.conf
daemonize yes
bind 192.168.200.129
requirepass 123456
dir /opt/master-slave/
port 6381
dbfilename dump_6381.rdb
pidfile /var/run/redis_6381.pid
logfile /opt/master-slave/6381.log
#用来指定主机：slaveof 主机ip 端口
slaveof 192.168.200.129 6379
#主机的密码
masterauth 123456
```

6. 启动master

```
redis-server /opt/master-slave/redis-6379.conf
```

7. 启动slave1

```
redis-server /opt/master-slave/redis-6380.conf
```

8. 启动salve1

```
redis-server /opt/master-slave/redis-6381.conf
```

###### 10.3.3 主从复制原理

* slave启动成功连接到master后，会给master发送数据同步消息（发送sync命令）
* master接收到slave发来的数据同步消息后，把主服务器的数据进行持久化到rdb文件，同时会收集接收到的用于修改数据的命令，master将传rdb文件发送给你slave，完成一次完全同步
* 全量复制：而slave服务在接收到master发来的rdb文件后，将其存盘并加载到内存
* 增量复制：master继续将收集到的修改命令依次传给slave，完成同步
* 但是只要重新连接master，一次完全同步（全量复制）将会被自动执行

###### 10.3.4 小结

1. 主redis挂掉以后情况会如何？从机是上位还是原地待命

> 主机挂掉后，从机会待命，小弟还是小弟，会等着大哥恢复，不会篡位。

2. 从挂掉后又恢复了，会继续从主同步数据么

> 会的，当从重启之后，会继续将中间缺失的数据同步过来。

3. info Replication：查看主从复制信息

##### 10.4 哨兵模式（Sentinel）

###### 10.4.1 what is sentinel

> 反客为主的自动版，能够自动监控master是否发生故障，如果故障了会根据投票数从slave中挑选一个作为master，其他的slave会自动转向同步新的master，实现故障自动转义。

###### 10.4.2 原理

> sentinel会按照指定的频率给master发送ping请求，看看master是否还活着，若master在指定时间内未正常响应sentinel发送的ping请求，sentinel则认为master挂掉了，但是这种情况存在误判的可能，比如：可能master并没有挂，只是sentinel和master之间的网络不通导致，导致ping失败
>
> 为了避免误判，通常会启动多个sentinel，一般是奇数个，比如3个，那么可以指定当有多个sentinel都觉得master挂掉了，此时才断定master真的挂掉了，通常这个值设置为sentinel的一半，比如sentinel的数量是3个，那么这个量就可以设置为2个
>
> 当多个sentinel经过判定，断定master确实挂掉了，接下来sentinel会进行故障转移：会从slave中投票选出一个服务器，将其升级为新的主服务器， 并让失效主服务器的其他从服务器slaveof指向新的主服务器； 当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器

###### 10.4.3 How to do

1. 需求：配置1主2从3哨兵

> 下面我们来实现1主2从3个sentinel的配置，当从的挂掉之后，要求最少有2个sentinel认为主的挂掉了，才进行故障转移
>
> 为了方便，我们在一台机器上进行模拟，我的机器ip是：192.168.200.129，通过端口来区分6个不同的节点（1个master、2个slave、3个sentinel），节点配置信息如下

![](C:\study\mlog\picture\5.png)

2. 创建案例工作目录：sentinel

> 执行下面命令创建 /opt/sentinel 目录，本次所有操作，均在 sentinel 目录进行

```
# 方便演示，停止所有的redis
ps -ef | grep redis | awk -F" " '{print $2;}' | xargs kill -9
mkdir /opt/sentinel
cd /opt/sentinel/
```

3. 将redis.conf复制到sentinel目录

> redis.conf 是redis默认配置文件

```
cp /opt/redis-6.2.1/redis.conf /opt/sentinel/
```

4. 创建master的配置文件：redis-6379.conf

> 在/opt/sentinel目录创建 redis-6379.conf 文件，内容如下，注意 192.168.200.129 是这个测试机器的ip，大家需要替换为自己的

```
include /opt/sentinel/redis.conf
daemonize yes
bind 192.168.200.129
dir /opt/sentinel/
port 6379
dbfilename dump_6379.rdb
pidfile /var/run/redis_6379.pid
logfile "./6379.log
```

5. 创建slave1的配置文件：redis-6380.conf

```
include /opt/sentinel/redis.conf
daemonize yes
bind 192.168.200.129
dir /opt/sentinel/
port 6380
dbfilename dump_6380.rdb
pidfile /var/run/redis_6380.pid
logfile "./6380.log
```

6. 创建slave2的配置文件：redis-6381.conf

```
include /opt/sentinel/redis.conf
daemonize yes
bind 192.168.200.129
dir /opt/sentinel/
port 6381
dbfilename dump_6381.rdb
pidfile /var/run/redis_6381.pid
logfile "./6381.log
```

7. 启动master、slave1、slave2

```
redis-server /opt/sentinel/redis-6379.conf
redis-server /opt/sentinel/redis-6380.conf
redis-server /opt/sentinel/redis-6381.conf
```

8. 配置slave1为master的从库

(1) 执行下面命令，连接slave1

```
redis-cli -h 192.168.200.129 -p 6380
```

(2)执行下面命令，指定slave1的作为master的从机

```
slaveof 192.168.200.129 6379
```

9. 配置slave2为master从库

同8

10. 创建sentinel1的配置文件：sentinel-26379.conf

> 在/opt/sentinel目录创建 sentinel-26379.conf 文件，内容如下

```
# 配置文件目录
dir /opt/sentinel/
# 日志文件位置
logfile "./sentinel-26379.log"
# pid文件
pidfile /var/run/sentinel_26379.pid
# 是否后台运行
daemonize yes
# 端口
port 26379
# 监控主服务器master的名字：mymaster，IP：192.168.200.129，port：6379，最后的数字2表示当
Sentinel集群中有2个Sentinel认为master存在故障不可用，则进行自动故障转移
sentinel monitor mymaster 192.168.200.129 6379 2
# master响应超时时间（毫秒），Sentinel会向master发送ping来确认master，如果在20秒内，ping
不通master，则主观认为master不可用
sentinel down-after-milliseconds mymaster 60000
# 故障转移超时时间（毫秒），如果3分钟内没有完成故障转移操作，则视为转移失败
sentinel failover-timeout mymaster 180000
# 故障转移之后，进行新的主从复制，配置项指定了最多有多少个slave对新的master进行同步，那可以理
解为1是串行复制，大于1是并行复制
sentinel parallel-syncs mymaster 1
# 指定mymaster主的密码（没有就不指定）
# sentinel auth-pass mymaster 123456
```

11. 同10 配置sentinel-26380.conf sentinel-26381.conf

12. 启动三个sentinel

启动方式2个

```
方式1：redis-server sentinel.conf --sentinel
方式2：redis-sentinel sentinel.conf
```

```
/opt/redis-6.2.1/src/redis-sentinel /opt/sentinel/sentinel-26379.conf
/opt/redis-6.2.1/src/redis-sentinel /opt/sentinel/sentinel-26380.conf
/opt/redis-6.2.1/src/redis-sentinel /opt/sentinel/sentinel-26381.conf
```

13. 分别查看3个sentinel的信息

```
redis-cli -p sentinel的端口
info sentinel
```

14. 验证故障自动转移是否成功

(1) 在master中执行下面命令，停止master

```
192.168.200.129:6379> shutdown
```

(2) 等待2分钟，等待完成故障转移

sentinel中我们配置 down-after-milliseconds 的值是60秒，表示判断主机下线时间是60秒，所以我们等2分钟，让系统先自动完成故障转移

(3)查看slave1的主从信息

```
info replication
```

(4)查看slave2的主从信息

同上

15. 恢复旧的master自动俯首称臣