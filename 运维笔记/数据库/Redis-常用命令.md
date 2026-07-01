# Redis 常用命令速查

> 命令行工具：`redis-cli` | 配置文件：`redis.conf`  
> 官方文档：https://redis.io/commands/

---

## 一、连接与退出

```bash
# 本地连接
redis-cli
redis-cli -n 1              # 选择数据库 1

# 远程连接
redis-cli -h 192.168.1.100 -p 6379
redis-cli -h 192.168.1.100 -p 6379 -a 'password'

# 连接后认证
AUTH password

# 退出
exit
quit
```

---

## 二、服务器管理

```bash
# 查看信息
INFO                    # 全部信息
INFO server             # 服务器信息
INFO clients            # 客户端连接
INFO memory             # 内存使用
INFO stats              # 统计信息
INFO replication        # 复制信息
INFO keyspace           # 键空间统计

# 查看配置
CONFIG GET *
CONFIG GET maxmemory
CONFIG GET save

# 设置配置
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru

# 重写配置文件
CONFIG REWRITE

# 查看客户端列表
CLIENT LIST

# 杀掉客户端
CLIENT KILL 127.0.0.1:54321

# 数据库大小
DBSIZE

# 清空数据库
FLUSHDB                 # 清空当前库
FLUSHALL                # 清空所有库

# 持久化
SAVE                    # 同步保存 RDB
BGSAVE                  # 异步保存 RDB
LASTSAVE                # 上次保存时间

# 慢查询
SLOWLOG GET 10          # 最近 10 条慢查询
SLOWLOG LEN             # 慢查询数量
SLOWLOG RESET           # 重置慢查询

# 监控
MONITOR                 # 实时监控所有命令（生产慎用）
```

---

## 三、键 (Key) 操作

```bash
# 查看键
KEYS *                  # 所有键（生产慎用，用 SCAN 替代）
KEYS user:*             # 匹配模式
SCAN 0 MATCH user:* COUNT 100   # 渐进式扫描

# 键类型
TYPE mykey              # string / list / set / zset / hash / stream

# 键是否存在
EXISTS mykey
EXISTS key1 key2 key3   # 返回存在的数量

# 删除键
DEL mykey
DEL key1 key2 key3
UNLINK mykey            # 异步删除（非阻塞）

# 过期时间
EXPIRE mykey 60         # 60 秒后过期
EXPIRE mykey 3600       # 1 小时
PEXPIRE mykey 1500      # 1500 毫秒
EXPIREAT mykey 1700000000   # 指定 Unix 时间戳过期
TTL mykey               # 剩余秒数（-1 永不过期，-2 不存在）
PTTL mykey              # 剩余毫秒数
PERSIST mykey           # 移除过期时间

# 重命名
RENAME oldkey newkey
RENAMENX oldkey newkey  # 仅 newkey 不存在时重命名

# 随机键
RANDOMKEY

# 迁移
MOVE mykey 1            # 移动到数据库 1
DUMP mykey              # 序列化键值
RESTORE newkey 0 <value>  # 反序列化

# 排序
SORT mylist             # 排序列表
SORT mylist DESC LIMIT 0 10
```

---

## 四、字符串 (String)

```bash
# 设置
SET key value
SET key value EX 60     # 60 秒过期
SET key value NX        # 仅不存在时设置（分布式锁）
SET key value XX        # 仅存在时设置
SETNX key value         # 等同 SET NX
SETEX key 60 value      # 设置 + 过期时间

# 批量设置/获取
MSET key1 val1 key2 val2 key3 val3
MGET key1 key2 key3

# 获取
GET key
GETSET key newvalue     # 设置新值，返回旧值

# 追加
APPEND key "suffix"

# 长度
STRLEN key

# 数字操作
INCR key                # +1
INCRBY key 10           # +10
DECR key                # -1
DECRBY key 5            # -5
INCRBYFLOAT key 0.5     # 浮点数

# 子串
GETRANGE key 0 3        # 获取 [0,3] 子串
SETRANGE key 0 "new"    # 从偏移 0 开始替换
```

---

## 五、哈希 (Hash)

```bash
# 设置
HSET user:1 name "Alice" age 25 email "alice@example.com"
HMSET user:1 name "Bob" age 30    # 批量设置
HSETNX user:1 phone "123456"      # 仅字段不存在时设置

# 获取
HGET user:1 name
HMGET user:1 name age email
HGETALL user:1                     # 获取所有字段和值
HKEYS user:1                       # 所有字段名
HVALS user:1                       # 所有值
HLEN user:1                        # 字段数量

# 是否存在字段
HEXISTS user:1 name

# 数字操作
HINCRBY user:1 age 1
HINCRBYFLOAT user:1 score 0.5

# 删除字段
HDEL user:1 phone

# 扫描
HSCAN user:1 0 MATCH name* COUNT 10
```

---

## 六、列表 (List)

```bash
# 左/右压入
LPUSH mylist "a" "b" "c"      # 左侧压入 → c, b, a
RPUSH mylist "d" "e"           # 右侧压入 → c, b, a, d, e
LPUSHX mylist "x"              # 仅列表存在时压入
RPUSHX mylist "y"

# 左/右弹出
LPOP mylist                    # 左侧弹出
RPOP mylist                    # 右侧弹出
BLPOP mylist 10                # 阻塞弹出，超时 10 秒
BRPOP mylist 10

# 弹出并压入
RPOPLPUSH source dest          # 从 source 右侧弹出，压入 dest 左侧
BRPOPLPUSH source dest 10

# 长度
LLEN mylist

# 范围
LRANGE mylist 0 -1             # 获取全部
LRANGE mylist 0 9              # 前 10 个
LRANGE mylist -5 -1            # 后 5 个

# 索引
LINDEX mylist 0                # 第 0 个元素
LSET mylist 0 "newval"         # 设置第 0 个元素

# 插入
LINSERT mylist BEFORE "b" "x"  # 在 b 前插入 x
LINSERT mylist AFTER "b" "x"   # 在 b 后插入 x

# 删除
LREM mylist 2 "a"              # 删除最多 2 个 "a"
LREM mylist 0 "a"              # 删除所有 "a"
LREM mylist -1 "a"             # 从右侧删除 1 个
LTRIM mylist 0 99              # 保留 [0,99]，其余删除
```

---

## 七、集合 (Set)

```bash
# 添加/删除
SADD myset "a" "b" "c" "d"
SREM myset "b"

# 查询
SMEMBERS myset                 # 全部成员
SCARD myset                    # 成员数量
SISMEMBER myset "a"            # 是否在集合中
SRANDMEMBER myset              # 随机返回一个
SRANDMEMBER myset 3            # 随机返回 3 个

# 弹出
SPOP myset                     # 随机弹出 1 个
SPOP myset 3                   # 随机弹出 3 个

# 移动
SMOVE source dest "a"          # 从 source 移到 dest

# 集合运算
SINTER set1 set2               # 交集
SINTERSTORE dest set1 set2     # 交集存入 dest
SUNION set1 set2               # 并集
SUNIONSTORE dest set1 set2
SDIFF set1 set2                # 差集（set1 有 set2 无）
SDIFFSTORE dest set1 set2

# 扫描
SSCAN myset 0 MATCH a* COUNT 10
```

---

## 八、有序集合 (Sorted Set / ZSet)

```bash
# 添加
ZADD myzset 1 "a" 2 "b" 3 "c"
ZADD myzset NX 4 "d"          # 仅不存在时添加
ZADD myzset CH 5 "a"          # 更新 a 的分数并返回变更数

# 查询
ZRANGE myzset 0 -1                       # 按分数升序，全部
ZRANGE myzset 0 -1 WITHSCORES            # 带分数
ZREVRANGE myzset 0 -1 WITHSCORES         # 降序
ZRANGEBYSCORE myzset 1 3                 # 分数在 [1,3]
ZRANGEBYSCORE myzset (1 3                # 分数在 (1,3]
ZCOUNT myzset 1 3                        # 分数范围内数量
ZCARD myzset                             # 成员数量
ZSCORE myzset "a"                        # 获取分数
ZRANK myzset "a"                         # 升序排名（0-based）
ZREVRANK myzset "a"                      # 降序排名

# 删除
ZREM myzset "a" "b"
ZREMRANGEBYRANK myzset 0 2              # 删除排名 0-2
ZREMRANGEBYSCORE myzset 1 3             # 删除分数 1-3

# 分数操作
ZINCRBY myzset 5 "a"                    # 分数 +5

# 集合运算
ZINTERSTORE dest 2 zset1 zset2 WEIGHTS 1 2  # 交集（加权）
ZUNIONSTORE dest 2 zset1 zset2               # 并集

# 扫描
ZSCAN myzset 0 MATCH a* COUNT 10
```

---

## 九、发布订阅 (Pub/Sub)

```bash
# 发布
PUBLISH channel "Hello World"

# 订阅
SUBSCRIBE channel1 channel2
PSUBSCRIBE news.*          # 按模式订阅

# 取消订阅
UNSUBSCRIBE channel1
PUNSUBSCRIBE news.*

# 查看
PUBSUB CHANNELS             # 活跃频道
PUBSUB NUMSUB channel1      # 订阅者数量
PUBSUB NUMPAT               # 模式订阅数量
```

---

## 十、事务 (Transactions)

```bash
# 事务
MULTI                       # 开始事务
SET key1 val1
SET key2 val2
EXEC                        # 执行事务
# 或
DISCARD                     # 放弃事务

# 乐观锁
WATCH key1 key2             # 监控键
MULTI
SET key1 newval
EXEC                        # 如果 key1/key2 被其他客户端修改，EXEC 返回 nil

UNWATCH                     # 取消监控
```

---

## 十一、Lua 脚本

```bash
# 执行脚本
EVAL "return redis.call('GET', KEYS[1])" 1 mykey
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey "newvalue"

# 脚本缓存
SCRIPT LOAD "return redis.call('GET', KEYS[1])"   # 返回 SHA
EVALSHA <sha> 1 mykey

SCRIPT EXISTS <sha>
SCRIPT FLUSH                                       # 清除所有脚本缓存
SCRIPT KILL                                        # 杀死运行中的脚本
```

---

## 十二、BitMap 和 HyperLogLog

```bash
# BitMap
SETBIT signin:2024:01 100 1          # 用户 100 签到
GETBIT signin:2024:01 100
BITCOUNT signin:2024:01              # 签到人数
BITPOS signin:2024:01 1              # 第一个签到用户
BITOP AND dest key1 key2             # 位运算 AND/OR/XOR/NOT

# HyperLogLog
PFADD visitors "user1" "user2" "user3"
PFCOUNT visitors                      # 近似去重计数
PFMERGE dest source1 source2          # 合并
```

---

## 十三、Stream (消息队列)

```bash
# 添加消息
XADD mystream * field1 value1 field2 value2
XADD mystream MAXLEN ~ 1000 * field value  # 限制长度

# 读取
XREAD COUNT 2 STREAMS mystream 0           # 从开头读
XREAD COUNT 2 BLOCK 5000 STREAMS mystream $  # 阻塞读新消息

# 消费者组
XGROUP CREATE mystream mygroup $ MKSTREAM   # 创建消费者组
XREADGROUP GROUP mygroup consumer1 COUNT 2 STREAMS mystream >   # 读取新消息
XACK mystream mygroup <msg-id>               # 确认消息
XREADGROUP GROUP mygroup consumer1 STREAMS mystream 0  # 读取待处理消息

# 查询
XLEN mystream
XRANGE mystream - + COUNT 10
XREVRANGE mystream + - COUNT 10
XDEL mystream <msg-id>

# 消费者组信息
XINFO STREAM mystream
XINFO GROUPS mystream
XPENDING mystream mygroup
```

---

## 十四、持久化配置

```bash
# RDB 快照配置
CONFIG SET save "900 1 300 10 60 10000"   # 900s 内 1 次修改, 300s 内 10 次...
CONFIG SET dbfilename dump.rdb
CONFIG SET dir /data/redis

# AOF 配置
CONFIG SET appendonly yes
CONFIG SET appendfsync everysec            # always / everysec / no
CONFIG SET auto-aof-rewrite-percentage 100
CONFIG SET auto-aof-rewrite-min-size 64mb

# 触发 AOF 重写
BGREWRITEAOF
```

---

## 十五、主从复制 / Sentinel

```bash
# 从库配置
REPLICAOF 192.168.1.100 6379
REPLICAOF NO ONE                    # 晋升为主库

# 复制状态
INFO replication
ROLE

# Sentinel 管理
SENTINEL masters
SENTINEL slaves mymaster
SENTINEL get-master-addr-by-name mymaster
SENTINEL failover mymaster
SENTINEL reset mymaster
```

---

## 十六、集群 (Cluster)

```bash
# 集群信息
CLUSTER INFO
CLUSTER NODES
CLUSTER SLOTS
CLUSTER KEYSLOT mykey               # 查看键的槽位

# 手动故障转移
CLUSTER FAILOVER
CLUSTER FAILOVER FORCE

# 重新分片
CLUSTER ADDSLOTS <slot> ...
CLUSTER DELSLOTS <slot> ...
CLUSTER SETSLOT <slot> NODE <node-id>
CLUSTER SETSLOT <slot> MIGRATING <node-id>
CLUSTER SETSLOT <slot> IMPORTING <node-id>

# 创建集群（命令行）
redis-cli --cluster create 192.168.1.1:6379 192.168.1.2:6379 192.168.1.3:6379 \
  192.168.1.4:6379 192.168.1.5:6379 192.168.1.6:6379 \
  --cluster-replicas 1

# 添加节点
redis-cli --cluster add-node 192.168.1.7:6379 192.168.1.1:6379

# 重新分片
redis-cli --cluster reshard 192.168.1.1:6379
```

---

## 十七、Geo（地理位置）

```bash
GEOADD cities 116.4074 39.9042 "Beijing" 121.4737 31.2304 "Shanghai"
GEOPOS cities "Beijing" "Shanghai"
GEODIST cities "Beijing" "Shanghai" km
GEORADIUS cities 116.0 39.0 500 km      # 半径搜索
GEORADIUSBYMEMBER cities "Beijing" 500 km
GEOHASH cities "Beijing" "Shanghai"
```

---

## 十八、内存管理

```bash
# 内存策略
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru

# 策略选项
# noeviction         - 不淘汰，写入报错
# allkeys-lru        - 所有键 LRU
# volatile-lru       - 有过期时间的键 LRU
# allkeys-random     - 所有键随机
# volatile-random    - 有过期时间的键随机
# volatile-ttl       - 最快过期的键
# allkeys-lfu        - 所有键 LFU (4.0+)
# volatile-lfu       - 有过期时间的键 LFU (4.0+)

# 内存分析
MEMORY USAGE mykey
MEMORY STATS
MEMORY DOCTOR              # 内存诊断建议
MEMORY PURGE               # 释放碎片
```

---

## 十九、Docker 中的 Redis

```bash
# 启动
docker run -d --name redis \
  -p 6379:6379 \
  redis:7-alpine

# 带密码
docker run -d --name redis \
  -p 6379:6379 \
  redis:7-alpine redis-server --requirepass mypassword

# 挂载配置
docker run -d --name redis \
  -p 6379:6379 \
  -v /path/to/redis.conf:/usr/local/etc/redis/redis.conf \
  -v /data/redis:/data \
  redis:7-alpine redis-server /usr/local/etc/redis/redis.conf

# 连接
docker exec -it redis redis-cli
docker exec -it redis redis-cli -a password

# 备份
docker exec redis redis-cli SAVE
docker cp redis:/data/dump.rdb ./dump.rdb
```

---

## 二十、常用场景命令速查

```bash
# 分布式锁
SET lock:resource "unique_id" NX EX 30

# 计数器
INCR page:view:2024-01-01

# 限流（滑动窗口）
ZADD rate:user:123 <now_ms> <now_ms>
ZREMRANGEBYSCORE rate:user:123 0 <now_ms - 60000>
ZCARD rate:user:123
EXPIRE rate:user:123 60

# 排行榜
ZADD leaderboard 100 "player1" 200 "player2"
ZREVRANGE leaderboard 0 9 WITHSCORES

# 消息队列
XADD queue:task * task "send_email" user_id 123
XREADGROUP GROUP workers worker1 COUNT 1 BLOCK 5000 STREAMS queue:task >

# 用户签到
SETBIT signin:2024:01:01 100 1
BITCOUNT signin:2024:01:01

# 去重计数
PFADD uv:page:home "user1" "user2" "user3"
PFCOUNT uv:page:home

# 附近的人
GEOADD locations 116.4074 39.9042 "user1"
GEORADIUSBYMEMBER locations "user1" 10 km
```