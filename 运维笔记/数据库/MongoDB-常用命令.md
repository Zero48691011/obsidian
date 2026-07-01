# MongoDB 常用命令速查

> 命令行工具：`mongosh`（新版）、`mongo`（旧版） | 配置文件：`mongod.conf`  
> 官方文档：https://www.mongodb.com/docs/manual/reference/

---

## 一、连接与退出

```bash
# 本地连接
mongosh
mongosh mydb
mongosh -u myuser -p mypassword --authenticationDatabase admin

# 远程连接
mongosh "mongodb://192.168.1.100:27017"
mongosh "mongodb://user:***@host:27017/mydb?authSource=admin"

# 复制集连接
mongosh "mongodb://host1:27017,host2:27017,host3:27017/mydb?replicaSet=rs0"

# 退出
exit
quit()
```

---

## 二、数据库操作

```javascript
// 查看所有数据库
show dbs
db.adminCommand({ listDatabases: 1 })

// 切换/创建数据库（插入数据时才会创建）
use mydb

// 查看当前数据库
db.getName()
db

// 删除数据库
db.dropDatabase()

// 查看数据库统计
db.stats()
db.stats(1024)    // 以 KB 为单位
```

---

## 三、集合操作

```javascript
// 创建集合
db.createCollection("users")
db.createCollection("logs", {
  capped: true,
  size: 10485760,        // 10MB
  max: 5000              // 最多 5000 条
})

// 查看所有集合
show collections
db.getCollectionNames()

// 删除集合
db.users.drop()

// 重命名集合
db.users.renameCollection("customers")

// 查看集合统计
db.users.stats()

// 查看集合大小
db.users.totalSize()
db.users.dataSize()
```

---

## 四、文档 CRUD — 增

```javascript
// 插入单条
db.users.insertOne({
  name: "Alice",
  email: "alice@example.com",
  age: 25,
  tags: ["developer", "admin"],
  address: { city: "Beijing", street: "Chang'an Ave" },
  createdAt: new Date()
})

// 插入多条
db.users.insertMany([
  { name: "Bob", email: "bob@example.com", age: 30 },
  { name: "Charlie", email: "charlie@example.com", age: 28 }
])

// 带选项
db.users.insertOne(
  { name: "Dave" },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
)

// 插入或替换
db.users.replaceOne(
  { name: "Alice" },
  { name: "Alice", email: "new@example.com", age: 26 },
  { upsert: true }
)
```

---

## 五、文档 CRUD — 查

```javascript
// 查询所有
db.users.find()
db.users.find().pretty()

// 精确匹配
db.users.find({ name: "Alice" })
db.users.find({ age: 25 })

// 比较运算符
db.users.find({ age: { $gt: 18 } })       // >
db.users.find({ age: { $gte: 25 } })      // >=
db.users.find({ age: { $lt: 30 } })       // <
db.users.find({ age: { $lte: 30 } })      // <=
db.users.find({ age: { $ne: 25 } })       // !=
db.users.find({ age: { $in: [25, 30] } })  // 在范围内
db.users.find({ age: { $nin: [25, 30] } }) // 不在范围内

// 逻辑运算符
db.users.find({ $and: [{ age: { $gte: 25 } }, { status: "active" }] })
db.users.find({ $or: [{ age: 25 }, { age: 30 }] })
db.users.find({ $not: { age: { $gte: 30 } } })
db.users.find({ $nor: [{ age: 25 }, { status: "banned" }] })

// 字段存在
db.users.find({ phone: { $exists: true } })
db.users.find({ phone: { $exists: false } })

// 类型匹配
db.users.find({ age: { $type: "int" } })

// 数组查询
db.users.find({ tags: "developer" })              // 包含 "developer"
db.users.find({ tags: { $all: ["a", "b"] } })     // 同时包含
db.users.find({ tags: { $size: 2 } })             // 数组长度
db.users.find({ "tags.0": "developer" })          // 按索引
db.users.find({ tags: { $elemMatch: { $gt: 5 } } }) // 元素匹配

// 嵌套文档
db.users.find({ "address.city": "Beijing" })

// 正则
db.users.find({ name: /^A/ })
db.users.find({ name: { $regex: /son$/, $options: "i" } })

// 投影
db.users.find({}, { name: 1, email: 1, _id: 0 })  // 只返回指定字段
db.users.find({}, { password: 0 })                 // 排除密码

// 排序
db.users.find().sort({ age: 1 })          // 升序
db.users.find().sort({ age: -1 })         // 降序
db.users.find().sort({ age: -1, name: 1 })

// 分页
db.users.find().limit(10)
db.users.find().skip(20).limit(10)

// 计数
db.users.countDocuments({ age: { $gte: 25 } })
db.users.estimatedDocumentCount()         // 估算，比 countDocuments 快

// 去重
db.users.distinct("status")
db.users.distinct("tags", { age: { $gte: 25 } })

// 查询单条
db.users.findOne({ name: "Alice" })
```

---

## 六、文档 CRUD — 改

```javascript
// 更新单条
db.users.updateOne(
  { name: "Alice" },
  { $set: { age: 26, email: "new@example.com" } }
)

// 更新多条
db.users.updateMany(
  { status: "pending" },
  { $set: { status: "active" } }
)

// 替换整条文档
db.users.replaceOne(
  { name: "Alice" },
  { name: "Alice", age: 27, email: "alice@example.com" }
)

// upsert（存在则更新，不存在则插入）
db.users.updateOne(
  { email: "eve@example.com" },
  { $set: { name: "Eve", age: 22 } },
  { upsert: true }
)

// 常用更新操作符
db.users.updateOne({ name: "Alice" }, { $inc: { age: 1 } })         // 自增
db.users.updateOne({ name: "Alice" }, { $mul: { score: 1.5 } })     // 乘法
db.users.updateOne({ name: "Alice" }, { $rename: { "name": "fullName" } }) // 重命名字段
db.users.updateOne({ name: "Alice" }, { $unset: { phone: "" } })    // 删除字段
db.users.updateOne({ name: "Alice" }, { $set: { updatedAt: new Date() } })
db.users.updateOne({ name: "Alice" }, { $currentDate: { updatedAt: true } })  // 设为当前时间
db.users.updateOne({ name: "Alice" }, { $min: { age: 20 } })        // 仅当新值小于当前值
db.users.updateOne({ name: "Alice" }, { $max: { score: 100 } })     // 仅当新值大于当前值

// 数组操作
db.users.updateOne({ name: "Alice" }, { $push: { tags: "newtag" } })       // 追加
db.users.updateOne({ name: "Alice" }, { $push: { tags: { $each: ["a","b"] } } }) // 批量追加
db.users.updateOne({ name: "Alice" }, { $addToSet: { tags: "developer" } }) // 去重追加
db.users.updateOne({ name: "Alice" }, { $pop: { tags: 1 } })               // 移除最后一个
db.users.updateOne({ name: "Alice" }, { $pop: { tags: -1 } })              // 移除第一个
db.users.updateOne({ name: "Alice" }, { $pull: { tags: "deprecated" } })   // 移除匹配元素
db.users.updateOne({ name: "Alice" }, { $pullAll: { tags: ["a","b"] } })   // 移除多个
```

---

## 七、文档 CRUD — 删

```javascript
// 删除单条
db.users.deleteOne({ name: "Alice" })

// 删除多条
db.users.deleteMany({ status: "banned" })
db.users.deleteMany({ createdAt: { $lt: new Date("2020-01-01") } })

// 删除所有
db.users.deleteMany({})
db.users.drop()    // 删除集合更快
```

---

## 八、聚合管道 (Aggregation)

```javascript
// 基础聚合
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$userId", total: { $sum: "$amount" }, count: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
])

// 常用操作符
// $match    - 过滤
// $group    - 分组
// $sort     - 排序
// $limit    - 限制
// $skip     - 跳过
// $project  - 投影
// $unwind   - 展开数组
// $lookup   - 左连接（类似 SQL JOIN）
// $addFields - 添加字段
// $count    - 计数
// $bucket   - 分桶
// $facet    - 多维度聚合

// $lookup 连接
db.orders.aggregate([
  { $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
  }},
  { $unwind: "$user" },
  { $project: { "user.name": 1, amount: 1 } }
])

// $unwind 展开数组
db.users.aggregate([
  { $unwind: "$tags" },
  { $group: { _id: "$tags", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])

// 日期分组
db.orders.aggregate([
  { $group: {
      _id: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } },
      total: { $sum: "$amount" },
      count: { $sum: 1 }
  }},
  { $sort: { _id: 1 } }
])

// 带条件的聚合
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: {
      _id: "$userId",
      total: { $sum: "$amount" },
      avg: { $avg: "$amount" },
      min: { $min: "$amount" },
      max: { $max: "$amount" },
      first: { $first: "$amount" },
      last: { $last: "$amount" }
  }},
  { $match: { total: { $gte: 1000 } } }
])
```

---

## 九、索引

```javascript
// 创建索引
db.users.createIndex({ name: 1 })                    // 升序
db.users.createIndex({ email: 1 }, { unique: true }) // 唯一索引
db.users.createIndex({ name: 1, age: -1 })           // 复合索引
db.users.createIndex({ tags: 1 })                    // 数组索引

// 文本索引
db.articles.createIndex({ title: "text", body: "text" })

// 地理空间索引
db.places.createIndex({ location: "2dsphere" })

// TTL 索引（自动过期）
db.logs.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })

// 部分索引
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { email: { $exists: true } } }
)

// 后台创建（不阻塞读写）
db.users.createIndex({ name: 1 }, { background: true })

// 查看索引
db.users.getIndexes()
db.users.getIndexKeys()

// 查看索引大小
db.users.totalIndexSize()

// 删除索引
db.users.dropIndex("name_1")
db.users.dropIndexes()              // 删除所有非 _id 索引

// 隐藏/取消隐藏索引
db.users.hideIndex("name_1")
db.users.unhideIndex("name_1")

// 查看查询计划
db.users.find({ name: "Alice" }).explain()
db.users.find({ name: "Alice" }).explain("executionStats")
```

---

## 十、用户与权限

```javascript
// 切换认证库
use admin

// 创建用户
db.createUser({
  user: "myuser",
  pwd: "mypassword",
  roles: [
    { role: "readWrite", db: "mydb" },
    { role: "read", db: "reporting" }
  ]
})

// 创建管理员
db.createUser({
  user: "admin",
  pwd: "adminpassword",
  roles: ["root"]
})

// 修改密码
db.changeUserPassword("myuser", "newpassword")

// 更新角色
db.updateUser("myuser", {
  roles: [{ role: "readWrite", db: "mydb" }]
})

// 查看用户
show users
db.getUsers()

// 查看用户角色
db.getUser("myuser")

// 删除用户
db.dropUser("myuser")

// 授予角色
db.grantRolesToUser("myuser", [{ role: "readWrite", db: "anotherdb" }])

// 撤销角色
db.revokeRolesFromUser("myuser", [{ role: "readWrite", db: "anotherdb" }])

// 内置角色
// read, readWrite, dbAdmin, userAdmin, clusterAdmin, root
// dbOwner, readAnyDatabase, readWriteAnyDatabase, userAdminAnyDatabase
```

---

## 十一、备份与恢复

```bash
# 备份（mongodump）
mongodump --db mydb --out /backup/mydb
mongodump --db mydb --collection users --out /backup
mongodump --uri "mongodb://user:***@host:27017" --db mydb --out /backup

# 备份所有数据库
mongodump --out /backup/all

# 恢复（mongorestore）
mongorestore --db mydb /backup/mydb
mongorestore --db mydb --collection users /backup/mydb/users.bson
mongorestore --drop /backup/mydb      # 恢复前删除已有数据

# 导出 JSON/CSV（mongoexport）
mongoexport --db mydb --collection users --out users.json
mongoexport --db mydb --collection users --type=csv --fields name,email,age --out users.csv
mongoexport --db mydb --collection users --query '{"age": {"$gte": 25}}' --out result.json

# 导入 JSON/CSV（mongoimport）
mongoimport --db mydb --collection users --file users.json
mongoimport --db mydb --collection users --type=csv --headerline --file users.csv
mongoimport --db mydb --collection users --jsonArray --file users.json --drop
```

---

## 十二、复制集 (Replica Set)

```javascript
// 查看复制集状态
rs.status()
rs.conf()
rs.isMaster()

// 初始化复制集
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "host1:27017", priority: 2 },
    { _id: 1, host: "host2:27017", priority: 1 },
    { _id: 2, host: "host3:27017", priority: 0, hidden: true }  // 隐藏节点
  ]
})

// 添加/移除节点
rs.add("host4:27017")
rs.addArb("host5:27017")    // 添加仲裁节点
rs.remove("host4:27017")

// 重新配置
rs.reconfig(config)

// 降级主节点
rs.stepDown()

// 查看复制延迟
rs.printSecondaryReplicationInfo()
rs.printReplicationInfo()

// 查看 oplog
use local
db.oplog.rs.find().sort({ $natural: -1 }).limit(1).pretty()
```

---

## 十三、分片 (Sharding)

```javascript
// 1. 在 Config Server 上初始化
rs.initiate({ _id: "cfgrs", configsvr: true, members: [{ _id: 0, host: "cfg1:27019" }] })

// 2. 在 mongos 上添加分片
sh.addShard("shard1/host1:27018,host2:27018")
sh.addShard("shard2/host3:27018,host4:27018")

// 3. 启用分片
sh.enableSharding("mydb")

// 4. 对集合分片
sh.shardCollection("mydb.users", { _id: "hashed" })           // 哈希分片
sh.shardCollection("mydb.orders", { userId: 1, createdAt: 1 }) // 范围分片

// 查看分片状态
sh.status()
sh.status({ verbose: true })

// 查看分片分布
db.users.getShardDistribution()

// 均衡器
sh.enableBalancing("mydb.users")
sh.disableBalancing("mydb.users")
sh.isBalancerRunning()
```

---

## 十四、性能与监控

```javascript
// 查看当前操作
db.currentOp()
db.currentOp({ active: true, secs_running: { $gt: 5 } })  // 超过 5 秒的操作

// 杀掉操作
db.killOp(opid)

// 查看服务器状态
db.serverStatus()
db.serverStatus().connections
db.serverStatus().opcounters

// 查看慢查询
db.getProfilingStatus()
db.setProfilingLevel(1)           // 记录慢查询（>100ms）
db.setProfilingLevel(2)           // 记录所有查询
db.setProfilingLevel(0)           // 关闭

// 查看慢查询日志
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty()

// 集合统计
db.users.stats()

// 有效大小
db.users.dataSize()
db.users.storageSize()
db.users.totalSize()              // 数据 + 索引

// 查看锁
db.serverStatus().locks

// 连接统计
db.serverStatus().connections
```

---

## 十五、事务

```javascript
// 开始会话
const session = db.getMongo().startSession()
session.startTransaction()

try {
  session.getDatabase("mydb").users.updateOne(
    { name: "Alice" },
    { $inc: { balance: -100 } }
  )
  session.getDatabase("mydb").users.updateOne(
    { name: "Bob" },
    { $inc: { balance: 100 } }
  )
  session.commitTransaction()
} catch (e) {
  session.abortTransaction()
} finally {
  session.endSession()
}

// 使用 withTransaction
session.withTransaction(async () => {
  await session.getDatabase("mydb").users.updateOne(
    { name: "Alice" }, { $inc: { balance: -100 } }
  )
  await session.getDatabase("mydb").users.updateOne(
    { name: "Bob" }, { $inc: { balance: 100 } }
  )
})
```

---

## 十六、数据验证 (Schema Validation)

```javascript
// 创建带验证的集合
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email"],
      properties: {
        name: {
          bsonType: "string",
          description: "must be a string"
        },
        email: {
          bsonType: "string",
          pattern: "^.+@.+\\..+$"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        },
        status: {
          enum: ["active", "inactive", "banned"]
        }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
})

// 修改验证规则
db.runCommand({
  collMod: "users",
  validator: { ... },
  validationLevel: "moderate"
})
```

---

## 十七、Docker 中的 MongoDB

```bash
# 启动单节点
docker run -d --name mongo \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=*** \
  -p 27017:27017 \
  mongo:7

# 复制集
docker run -d --name mongo1 \
  -p 27017:27017 \
  mongo:7 mongod --replSet rs0

# 连接
docker exec -it mongo mongosh -u admin -p ***
docker exec -it mongo mongosh "mongodb://admin:***@localhost:27017"

# 备份
docker exec mongo mongodump --db mydb --out /backup
docker cp mongo:/backup ./backup

# 恢复
docker cp ./backup mongo:/backup
docker exec mongo mongorestore --db mydb /backup/mydb
```

---

## 十八、常用查询速查

```javascript
// 分页
db.users.find().sort({ _id: -1 }).skip(20).limit(10)

// 模糊搜索
db.users.find({ name: /alice/i })

// 范围内搜索
db.users.find({ age: { $gte: 18, $lte: 60 } })

// 空值查询
db.users.find({ phone: null })
db.users.find({ phone: { $exists: false } })

// 数组不为空
db.users.find({ tags: { $exists: true, $not: { $size: 0 } } })

// 子文档匹配
db.users.find({ "address.city": "Beijing" })

// 随机取一条
db.users.aggregate([{ $sample: { size: 1 } }])

// 分组 Top N
db.orders.aggregate([
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
])

// 更新嵌套字段
db.users.updateOne(
  { name: "Alice" },
  { $set: { "address.city": "Shanghai" } }
)

// 批量写入
db.users.bulkWrite([
  { insertOne: { document: { name: "A" } } },
  { updateOne: { filter: { name: "B" }, update: { $set: { age: 30 } } } },
  { deleteOne: { filter: { name: "C" } } },
  { replaceOne: { filter: { name: "D" }, replacement: { name: "D2" } } }
])
```