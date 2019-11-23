## linux操作日志、配置文件好用命令

以下命令对批量操作日志文件、配置文件有较大帮助

- Linux中的ps命令是Process Status的缩写。**ps aux 和ps -ef** 两者展示风格不同
- 管道操作符 | 可将指令连接起来，前一个指令的输出作物后一个指令的输入
- find文件检索 find / -name "target3.java" 在根目录下找
- grep文件内容检索，-v反转查找
- vim编辑文件，按i进入编辑模式。：wq保存退出
- cat命令（concatenate） 显示文件内容，比如几个文件连接显示。 cat+'>'复制
- sed批量替换文本  -i 表示 inplace edit， 就地修改文件。`"s/原字符串/新字符串/g" 后面的g表示对全局进行替换
- awk对表格做统计

```
sed "s/6380/6381/g" redis-6380.conf > redis-6381.conf
cat redis-6380.conf | grep -v "#" | grep -v "^$"
cat sentinel26379.conf | grep -v "#" | grep -v "^$" > sentinel-26379.conf
ps -ef | grep redis-server 
kill -9
```







## redis主从复制



部署中常用命令

```

kill -9 22284

redis-server redis-6379.conf
redis-server redis-6380.conf
redis-server redis-6381.conf

cat config/redis-6380.conf | grep -v "#" | grep -v "^$"
```



cat config/redis-6380.conf | grep -v "#" | grep -v "^$"



### 步骤一：以后台的方式启动redis6379

```
去掉配置文件查看6379
cat redis-6379.conf | grep -v "#" | grep -v "^$"
redis-server redis-6379.conf
查看进程
ps -ef | grep redis-server 
按端口启动客户端
redis-cli -p 6379
查看信息
info

发现角色是主结点
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online,offset=1288,lag=1
master_replid:fb95170e884c6c842809556d35c4b945ac2d81ba
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:1288
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:1288
```



### 步骤二：复制配置文件，批量修改端口，并配置为从结点



```
批量替换端口，并拷贝一份，记得修改为slave结点
sed "s/6379/6380/g" redis-6379.conf > redis-6380.conf
vim redis-6380.conf

按端口启动客户端
redis-cli -p 6379
查看信息
info

启动从结点，进入主结点客户端查看信息发现已经有一个slave
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online,offset=1288,lag=1
master_replid:fb95170e884c6c842809556d35c4b945ac2d81ba
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:1288
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:1288

复制从结点信息，只需要批量修改端口就够了
sed "s/6380/6381/g" redis-6380.conf > redis-6381.conf



查看进程发现三个redis结点都启动了
ps -ef | grep redis-server

检查一下发现6379已经有三个从结点
redis-cli -p 6379
info replication
```



### 步骤三：以后台的方式启动redis6380、redis6381。



## 哨兵部署

### 步骤一，修改第一个哨兵配置文件

```
cp sentinel.conf config/sentinel-26379.conf
vim sentinel-26379.conf
打开哨兵配置文件改成这个样子
port 26379
dir /root/redis/data
logfile "26379.log"
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000

```



### 步骤二，启动并查看第一个哨兵配置文件

```
启动,&意思是以后台方式启动
redis-sentinel sentinel26379.conf&
ps -ef | grep redis-sentinel
也可以看看信息
redis-cli -p 26379
127.0.0.1:26379> info
可以看到
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=127.0.0.1:6379,slaves=2,sentinels=1
```



### 步骤三，为其他两个哨兵配置文件

```
剩下的两个哨兵 批量修改端口就行了 
sed "s/26379/26380/g" sentinel-26379.conf > sentinel-26380.conf
sed "s/26380/26381/g" sentinel-26380.conf > sentinel-26381.conf
```



```
redis-sentinel sentinel-26379.conf&
redis-sentinel sentinel-26380.conf&
redis-sentinel sentinel-26381.conf&

查看
root@iZ2zeczmnls09fofwrs84gZ:~/redis/config# ps -ef | grep redis-sentinel
root     22513 22452  0 16:17 pts/1    00:00:00 redis-sentinel *:26379 [sentinel]
root     22522 22452  0 16:24 pts/1    00:00:00 redis-sentinel *:26380 [sentinel]
root     22526 22452  0 16:24 pts/1    00:00:00 redis-sentinel *:26381 [sentinel]
root     22531 22452  0 16:24 pts/1    00:00:00 grep --color=auto redis-sentinel
```

## 故障转移实践

```


查一下master的进程id
ps -ef | grep redis-server
发现id是22284，杀掉进程
kill -9 22284
cat sentinel26379.conf | grep -v "#" | grep -v "^$"
```



启动redis

### 哨兵日志信息

```



22522:X 21 Aug 16:24:33.359 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo

22522:X 21 Aug 16:24:33.359 # Redis version=4.0.8, bits=64, commit=00000000, modified=0, pid=22522, just started

22522:X 21 Aug 16:24:33.359 # Configuration loaded

22522:X 21 Aug 16:24:33.361 * Running mode=sentinel, port=26380.

22522:X 21 Aug 16:24:33.361 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.

22522:X 21 Aug 16:24:33.361 # Sentinel ID is 7c4d352d8734c7770ac5fcc3efef06f7876b3caf

22522:X 21 Aug 16:24:33.361 # +monitor master mymaster 127.0.0.1 6379 quorum 2

22522:X 21 Aug 16:40:38.162 # +sdown master mymaster 127.0.0.1 6379

22522:X 21 Aug 16:54:11.385 * +reboot master mymaster 127.0.0.1 6379

22522:X 21 Aug 16:54:11.474 # -sdown master mymaster 127.0.0.1 6379

22522:X 21 Aug 16:54:22.224 * +reboot master mymaster 127.0.0.1 6379

22742:X 21 Aug 17:30:15.923 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo

22742:X 21 Aug 17:30:15.923 # Redis version=4.0.8, bits=64, commit=00000000, modified=0, pid=22742, just started

22742:X 21 Aug 17:30:15.923 # Configuration loaded

22742:X 21 Aug 17:30:15.925 * Running mode=sentinel, port=26380.

22742:X 21 Aug 17:30:15.925 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.

22742:X 21 Aug 17:30:15.925 # Sentinel ID is 7c4d352d8734c7770ac5fcc3efef06f7876b3caf

22742:X 21 Aug 17:30:15.925 # +monitor master mymaster 127.0.0.1 6379 quorum 2

22742:X 21 Aug 17:32:32.701 # +sdown master mymaster 127.0.0.1 6379

22742:X 21 Aug 17:37:09.398 * +reboot master mymaster 127.0.0.1 6379

22742:X 21 Aug 17:37:09.460 # -sdown master mymaster 127.0.0.1 6379

22742:X 21 Aug 17:37:29.509 * +reboot master mymaster 127.0.0.1 6379

22742:X 21 Aug 17:47:13.089 # +sdown master mymaster 127.0.0.1 6379 主观下线

1002:X 21 Aug 19:17:29.277 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo

1002:X 21 Aug 19:17:29.277 # Redis version=4.0.8, bits=64, commit=00000000, modified=0, pid=1002, just started

1002:X 21 Aug 19:17:29.277 # Configuration loaded

1002:X 21 Aug 19:17:29.279 * Running mode=sentinel, port=26380.

1002:X 21 Aug 19:17:29.279 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.

1002:X 21 Aug 19:17:29.280 # Sentinel ID is 450f7f7ea410c39d41e974affc83cda0113994f7

1002:X 21 Aug 19:17:29.280 # +monitor master mymaster 127.0.0.1 6379 quorum 2

1002:X 21 Aug 19:17:29.281 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6379

1002:X 21 Aug 19:17:29.281 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379

1002:X 21 Aug 19:17:29.363 * +sentinel sentinel d192c1ac24df230470e06666d30414c1298e7479 127.0.0.1 26379 @ mymaster 127.0.0.1 6379

1002:X 21 Aug 19:17:35.772 * +sentinel sentinel 4cc8249091cded4f2eedeacc91fd42881c3a8f42 127.0.0.1 26381 @ mymaster 127.0.0.1 6379

1002:X 21 Aug 19:20:42.415 # +sdown master mymaster 127.0.0.1 6379

1002:X 21 Aug 19:20:42.516 # +new-epoch 1

1002:X 21 Aug 19:20:42.517 # +vote-for-leader d192c1ac24df230470e06666d30414c1298e7479 1

1002:X 21 Aug 19:20:43.517 # +odown master mymaster 127.0.0.1 6379 #quorum 3/2客观下线

1002:X 21 Aug 19:20:43.518 # Next failover delay: I will not start a failover before Wed Aug 21 19:26:43 2019

1002:X 21 Aug 19:20:43.759 # +config-update-from sentinel d192c1ac24df230470e06666d30414c1298e7479 127.0.0.1 26379 @ mymaster 127.0.0.1 6379

1002:X 21 Aug 19:20:43.759 # +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6381 最重要的标识 master切换

1002:X 21 Aug 19:20:43.759 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381

1002:X 21 Aug 19:20:43.759 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6381
 
```



### 查看6379旧主结点日志

```

998:X 21 Aug 19:20:42.431 # +sdown master mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:42.514 # +odown master mymaster 127.0.0.1 6379 #quorum 2/2

998:X 21 Aug 19:20:42.514 # +new-epoch 1

998:X 21 Aug 19:20:42.514 # +try-failover master mymaster 127.0.0.1 6379哨兵开始进行故障恢复

998:X 21 Aug 19:20:42.515 # +vote-for-leader d192c1ac24df230470e06666d30414c1298e7479 1

998:X 21 Aug 19:20:42.517 # 450f7f7ea410c39d41e974affc83cda0113994f7 voted for d192c1ac24df230470e06666d30414c1298e7479 1

998:X 21 Aug 19:20:42.517 # 4cc8249091cded4f2eedeacc91fd42881c3a8f42 voted for d192c1ac24df230470e06666d30414c1298e7479 1

998:X 21 Aug 19:20:42.573 # +elected-leader master mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:42.573 # +failover-state-select-slave master mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:42.673 # +selected-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:42.673 * +failover-state-send-slaveof-noone slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:42.728 * +failover-state-wait-promotion slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:43.702 # +promoted-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:43.702 # +failover-state-reconf-slaves master mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:43.758 * +slave-reconf-sent slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:44.646 # -odown master mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:44.764 * +slave-reconf-inprog slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:44.764 * +slave-reconf-done slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:44.830 # +failover-end master mymaster 127.0.0.1 6379

998:X 21 Aug 19:20:44.830 # +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6381

998:X 21 Aug 19:20:44.830 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381

998:X 21 Aug 19:20:44.830 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6381

998:X 21 Aug 19:21:14.877 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6381

```



### 查看6380新主结点日志

即被选举为从结点上升为新主结点的日志

```
从结点日志



989:M 21 Aug 19:20:42.728 * MASTER MODE enabled (user request from 'id=4 addr=127.0.0.1:34346 fd=8 name=sentinel-d192c1ac-cmd age=199 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=133 qbuf-free=32635 obl=36 oll=0 omem=0 events=r cmd=exec') 接受到一条mode请求，希望让自己成为新的master

989:M 21 Aug 19:20:42.729 # CONFIG REWRITE executed with success.配置重写

989:M 21 Aug 19:20:44.588 * Slave 127.0.0.1:6380 asks for synchronization 收到6380请求复制，去做bgsave

989:M 21 Aug 19:20:44.588 * Partial resynchronization request from 127.0.0.1:6380 accepted. Sending 422 bytes of backlog starting from offset 32580.
```

