---
title: MongoDB 详解
categories: 
- 后端
tags:
- 数据库
- MongoDB
- NoSQL
- 文档数据库
---

## 一、MongoDB 是什么

MongoDB 是一个**文档型** NoSQL 数据库，以 JSON 风格的 BSON 格式存储数据。它不要求预定义 Schema，支持嵌入式文档和数组，天然适合表达复杂的数据结构。<!--more-->

### 核心概念对比

| MySQL | MongoDB |
|-------|---------|
| 数据库（Database） | 数据库（Database） |
| 表（Table） | 集合（Collection） |
| 行（Row） | 文档（Document） |
| 列（Column） | 字段（Field） |
| 主键（Primary Key） | `_id`（自动生成 ObjectId） |
| 索引（Index） | 索引（Index） |
| JOIN | `$lookup` 聚合 / 嵌入式文档 |

```json
// MySQL 需两张表 JOIN
// users: id, name
// addresses: user_id, city, street

// MongoDB 可直接嵌入
{
  "_id": ObjectId("..."),
  "name": "Alice",
  "addresses": [
    { "city": "北京", "street": "长安街" },
    { "city": "上海", "street": "南京路" }
  ]
}
```

## 二、基本操作

### 连接

```bash
mongosh                                # 连接本地
mongosh "mongodb://localhost:27017"
show dbs;                              # 查看数据库
use myapp;                             # 切换/创建数据库
db;                                    # 查看当前数据库
```

### CRUD

```js
// 插入
db.users.insertOne({ name: "Alice", age: 25, email: "alice@example.com" });
db.users.insertMany([
  { name: "Bob", age: 30 },
  { name: "Charlie", age: 28, tags: ["dev", "design"] },
]);

// 查询
db.users.find();                                    // 全部
db.users.find({ age: { $gt: 25 } });               // age > 25
db.users.find({ age: { $gte: 20, $lte: 30 } });    // 20 ≤ age ≤ 30
db.users.find({ name: /^A/ });                     // 正则
db.users.find({ tags: "dev" });                    // 数组包含
db.users.find({}, { name: 1, email: 1 });           // 只返回指定字段（projection）
db.users.find().sort({ age: -1 }).limit(10).skip(0); // 排序 + 分页

// 查询单个
db.users.findOne({ email: "alice@example.com" });

// 更新
db.users.updateOne(
  { name: "Alice" },
  { $set: { age: 26 } }
);
db.users.updateMany(
  { age: { $lt: 20 } },
  { $set: { status: "minor" }, $inc: { age: 1 } }
);
db.users.replaceOne(
  { name: "Alice" },
  { name: "Alice", age: 26, email: "alice@new.com" }  // 整文档替换
);

// 删除
db.users.deleteOne({ name: "Charlie" });
db.users.deleteMany({ age: { $lt: 18 } });

// 计数
db.users.countDocuments({ age: { $gt: 20 } });
```

### 比较运算符

```js
$eq / $ne       等于 / 不等于
$gt / $gte      大于 / 大于等于
$lt / $lte      小于 / 小于等于
$in / $nin      在列表中 / 不在
$exists         字段是否存在
$type           字段类型
$regex          正则匹配
```

## 三、嵌入式文档与数组操作

```js
// 插入含嵌入文档和数组的数据
db.orders.insertOne({
  userId: ObjectId("..."),
  items: [
    { product: "手机", price: 5000, quantity: 1 },
    { product: "耳机", price: 200, quantity: 2 },
  ],
  shipping: { address: "北京", status: "pending" },
  total: 5400,
});

// 查询嵌套字段
db.orders.find({ "shipping.status": "pending" });
db.orders.find({ "items.price": { $gt: 1000 } });

// 更新嵌套字段
db.orders.updateOne(
  { _id: ObjectId("...") },
  { $set: { "shipping.status": "shipped" } }
);

// 数组更新
db.orders.updateOne(
  { _id: ObjectId("..."), "items.product": "手机" },
  { $inc: { "items.$.quantity": 1 } }      // $ 定位匹配的元素
);

db.orders.updateOne(
  { _id: ObjectId("...") },
  { $push: { items: { product: "充电器", price: 100, quantity: 1 } } }
);

db.orders.updateOne(
  { _id: ObjectId("...") },
  { $pull: { items: { product: "耳机" } } }
);
```

## 四、聚合管道（Aggregation Pipeline）

```js
db.orders.aggregate([
  // 第一阶段：过滤（类似 WHERE）
  { $match: { status: "completed" } },

  // 第二阶段：分组（类似 GROUP BY）
  { $group: {
    _id: "$userId",
    totalSpent: { $sum: "$total" },
    orderCount: { $sum: 1 },
    avgAmount: { $avg: "$total" },
  }},

  // 第三阶段：排序
  { $sort: { totalSpent: -1 } },

  // 第四阶段：限制
  { $limit: 10 },

  // 第五阶段：投影（选择返回字段）
  { $project: {
    _id: 0,
    userId: "$_id",
    totalSpent: 1,
    orderCount: 1,
  }},
]);
```

### 常用阶段

```text
$match     过滤文档（WHERE）
$project   字段选择/重命名（SELECT）
$group     分组聚合（GROUP BY）
$sort      排序（ORDER BY）
$limit     限制条数
$skip      跳过条数
$unwind    数组展开（将数组元素拆分为多个文档）
$lookup    左连接（JOIN）
$addFields 添加新字段
$bucket    分桶统计
$facet     多维度聚合
```

### $lookup 关联查询

```js
db.orders.aggregate([
  {
    $lookup: {
      from: "users",                        // 关联集合
      localField: "userId",                 // 本地字段
      foreignField: "_id",                  // 关联字段
      as: "user",                           // 结果字段名
    },
  },
  { $unwind: "$user" },                     // 展开数组
  {
    $project: {
      "user.name": 1,
      "user.email": 1,
      total: 1,
      items: 1,
    },
  },
]);
```

## 五、索引

```js
// 创建索引
db.users.createIndex({ email: 1 });                      // 单字段索引（1 升序，-1 降序）
db.users.createIndex({ name: 1, age: -1 });              // 复合索引
db.users.createIndex({ email: 1 }, { unique: true });    // 唯一索引
db.users.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 }); // TTL 索引（到期自动删除）

// 文本索引
db.articles.createIndex({ title: "text", content: "text" });
db.articles.find({ $text: { $search: "MongoDB" } });

// 查看索引
db.users.getIndexes();

// 删除索引
db.users.dropIndex("email_1");
```

### 索引策略

```js
// 复合索引的匹配规则
db.users.createIndex({ status: 1, age: -1, createdAt: 1 });

// 精确匹配 → 排序 → 范围查询
db.users.find({ status: "active" })               // ✅ 使用 status
  .sort({ age: -1 })                               // ✅ 使用 age（排序）
  .limit(20);
// 等价规则：等值字段在前，排序字段次之，范围字段最后
```

### explain

```js
db.users.find({ email: "alice@example.com" }).explain("executionStats");
```

```text
关键字段：
  stage        IXSCAN（索引扫描） / COLLSCAN（集合扫描） / FETCH（回表）
  nReturned   返回文档数
  totalDocsExamined  扫描文档数（越小越好）
  totalKeysExamined  扫描索引项数
```

## 六、Mongoose（Node.js ODM）

```bash
npm install mongoose
```

```js
const mongoose = require('mongoose');
mongoose.connect('mongodb://localhost:27017/myapp');

// 定义 Schema
const userSchema = new mongoose.Schema({
  name: { type: String, required: true, trim: true },
  email: { type: String, unique: true, lowercase: true },
  age: { type: Number, min: 0, max: 150, default: 0 },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  tags: [String],
  address: {
    city: String,
    street: String,
  },
}, {
  timestamps: true,   // 自动添加 createdAt / updatedAt
});

// 虚拟字段
userSchema.virtual('isAdult').get(function() {
  return this.age >= 18;
});

// 实例方法
userSchema.methods.toJSON = function() {
  const obj = this.toObject();
  delete obj.__v;
  return obj;
};

// 静态方法
userSchema.statics.findByEmail = function(email) {
  return this.findOne({ email });
};

// 创建 Model
const User = mongoose.model('User', userSchema);

// CRUD
const user = await User.create({ name: 'Alice', email: 'alice@example.com', age: 25 });
const users = await User.find({ age: { $gt: 20 } }).sort({ age: -1 }).limit(10);
const updated = await User.findByIdAndUpdate(user._id, { age: 26 }, { new: true });
await User.findByIdAndDelete(user._id);
```

## 七、复制集与分片

### 复制集（Replica Set）

```text
┌──────────┐       ┌──────────┐
│  PRIMARY  │──────→│ SECONDARY│
│（读写）   │       │（只读）   │
└──────────┘       └──────────┘
       │
       ↓
┌──────────┐
│  ARBITER │（仅投票）
└──────────┘

特性：
  ├── 主节点故障时自动切换（通常 10-30 秒）
  ├── 默认读写主节点，可配置读从节点
  ├── 数据通过 oplog 同步
  └── 建议至少 3 个节点
```

### 分片（Sharding）

```text
数据按分片键分布到多个节点上，对应用透明。

选择分片键的原则：
  ├── 基数大（唯一值多）
  ├── 读写均匀分布
  ├── 避免单调递增（否则所有写入集中在最后一个分片）
  └── 常见选择：userId、地域 hash
```

## 八、性能优化

```text
├── 索引：经常查询的字段建立索引
├── Projection：只返回需要的字段（.find({}, { name: 1 })）
├── 批量操作：用 bulkWrite 替代逐条 insert
├── 分页：用 _id 游标替代 skip（大分页时）
│   .find({ _id: { $gt: lastId } }).limit(20)
├── 避免负数查询（$ne / $nin / $not）
├── 索引覆盖：查询字段都在索引中
└── 连接数管理：用连接池（默认 100）
```

## 九、面试题

### Q1: MongoDB 和 MySQL 怎么选

```text
MongoDB 优势：
  Schema 灵活、嵌入式文档减少 JOIN、水平扩展原生支持

MySQL 优势：
  强事务 ACID、复杂 JOIN 查询、成熟生态

选 MongoDB：内容管理、日志、IoT、快速原型
选 MySQL：金融、订单、ERP、报表
```

### Q2: MongoDB 的事务支持

```text
MongoDB 4.0+ 支持多文档 ACID 事务，但：
  1. 只能在复制集上使用
  2. 性能开销比 MySQL 大
  3. 默认不开启，需显式使用 session

const session = mongoose.startSession();
session.startTransaction();
try {
  await User.create([{ name: "Alice" }], { session });
  await Order.create([{ userId: "xxx" }], { session });
  await session.commitTransaction();
} catch {
  await session.abortTransaction();
}
```

### Q3: ObjectId 的结构

```text
ObjectId("665f1a2b3c4d5e6f7a8b9c0d")
  └──────┬──────┘
  24 位十六进制 = 12 字节

  4 字节  ─ 时间戳（秒，从 Unix 纪元开始）
  5 字节  ─ 随机值（机器 + 进程 + 自增计数器）
  3 字节  ─ 自增计数器（递增）

  因此 ObjectId 本身包含创建时间 ≈ 隐含 createdAt
  db.users.find().sort({ _id: -1 })  ≈ 按创建时间降序
```

### Q4: 什么是 GridFS

```text
GridFS 是 MongoDB 的分布式文件存储方案。
文件超过 16MB（单个文档上限）时，自动切分为多个 255KB 的 chunk。
适合存储大文件（图片、音频、视频），但不建议替代专业 OSS。
```

### Q5: MongoDB 的写入安全级别

```js
// writeConcern 控制写入确认级别
{ w: 1 }            // 主节点确认即可（默认）
{ w: "majority" }   // 大多数节点确认（安全但慢）
{ w: 0 }            // 不确认（快但有风险）

// journal：是否写入日志后才确认
{ j: true }         // 写入 journal 后确认（持久性更高）
```
