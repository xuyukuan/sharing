# MongoDB

### MongoDB的优势
##### 灵活的数据模型
你想要的数据是什么样子，存在数据库里的就是什么样子。
可以在文档中直接插入数组之类的复杂数据类型，并且文档的key和value不是固定的数据类型和大小，所以开发者在使用MongoDB时无须预定义关系型数据库中的”表”等数据库对象，设计数据库将变得非常方便，可以大大地提升开发进度。
##### 易扩展, 高可用性
副本集， 分片
##### 大数据量，高性能
NoSQL数据库都具有非常高的读写性能，尤其在大数据量下，同样表现优秀。这得益于它的无关系性，数据库的结构简单。
#### 数据压缩
自从MongoDB 3.0推出以后，MongoDB引入了一个高性能的存储引擎WiredTiger，并且它在数据压缩性能上得到了极大的提升，跟之前的MMAP引擎相比，压缩比至少可增加5倍以上，可以极大地改善磁盘空间使用率。

### MongoDB不适用的应用场景
在某些场景下，MongoDB作为一个非关系型数据库有其局限性。MongoDB不支持事务操作，所以需要用到事务的应用建议不用MongoDB，另外MongoDB目前不支持join操作，需要复杂查询的应用也不建议使用MongoDB。

### Compare with RDBMS
|            RDBMS            |        MongoDB       |
|:---------------------------:|:--------------------:|
| database                    | database             |
| table                       | collection           |
| row                         | document             |
| column                      | field                |
| primary key                 | primary key(_id)     |
| aggregation (e.g. group by) | aggregation pipeline |

### Design a task tracking app
#### RDB
| id |  name |        email        |   title  |
|:--:|:-----:|:-------------------:|:--------:|
| 1  | Billy | Billy.Xu@xplusz.com | engineer |

| id |  street  |    city   | user_id |
|:--:|:--------:|:---------:|:-------:|
| 1  | street 1 | Su zhou   | 1       |
| 2  | street 2 | Shang hai | 1       |

#### MongoDB
```javascript
> db.user.findOne( { email: "Billy.Xu@xplusz.com"} ){   _id: 1,   name: "Billy",   email: "Billy.Xu@xplusz.com",   title: "engineer",   addresses: [      { street: "street 1", city: "Su zhou" },      { street: "street 2", city: "Shang hai" }   ]}
```

#### Embeding vs Referencing
* Embeding: one - many (few)

```javascript
> db.user.findOne( { email: "Billy.Xu@xplusz.com"} ){   _id: 1,   name: "Billy",   // ... previous fields
   tasks: [      {          summary: "Run 10 km",          due_date: ISODate("2018-12-25T08:37:50.465Z"),          status: "NOT_STARTED"		},      { // another task }   ]}
```
* Referencing: one - many

```javascript
> db.user.findOne( { email: "Billy.Xu@xplusz.com"} ){   _id: 1,   name: "Billy",   email: "Billy.Xu@xplusz.com",   title: "engineer",   addresses: [      { street: "street 1", city: "Su zhou" },      { street: "street 2", city: "Shang hai" }   ]}
```

```javascript
> db.task.findOne({user_id: 1}){	_id: 5,	summary: "Run 10 km",   due_date: ISODate(),   status: "NOT_STARTED",   user_id: 1}
```

* many to many

### Get you hands dirty
[Demo data file](https://github.com/ozlerhakan/mongodb-json-files/tree/master/datasets)
##### Import
```bash
mongorestore --drop --host=127.0.0.1 -d twitter-db -c tweets ./tweets.bson
```
##### Export

```bash
mongodump --host=127.0.0.1 -d twitter-db -c tweets -o .
```

### MongoDB basic query
##### 点赞数在1000和1100之间的记录
```javascript
db.getCollection('tweets').find({
    favorite_count: {
        $gt: 1000,
        $lt: 1100
    }
})
```

##### 含Javascript的记录
```javascript
db.getCollection('tweets').find({
    text: /Javascript/
})
```
```javascript
db.getCollection('tweets').find( {
	$text: { $search: "javascript" }
} )
```

##### 用户Premier League所发的所有记录
```javascript
db.getCollection('tweets').find({
    'user.name': 'Premier League'
})
```
##### 用户Premier League和BBC News (World)所发的被转发超过500次的记录
```javascript
db.getCollection('tweets').find({
    'user.name': {
        $in: ['Premier League', 'BBC News (World)']
    },
    retweet_count: { $gte: 500 }
}, {
    _id: 0,
    text: 1,
    'user.name': 1
})
```
##### 提到Allan Yeung的记录
```javascript
db.getCollection('tweets').find({
    'entities.user_mentions.name': 'Allan Yeung'
})
```

### Aggregation Framework
##### 按发tweet记录数对用户排序
```javascript
db.getCollection('tweets').aggregate([{
    $group: {
        _id: '$user.screen_name',
        count: { $sum: 1 }
    }
}, {
    $sort: {
        count: -1
    }
}])
```
##### 找出粉丝人数与朋友人数比例最大的用户
```javascript
db.getCollection('tweets').aggregate([{
    $match: {
        'user.friends_count': { $gt: 0 },
        'user.followers_count': { $gt: 0 }
    }
}, {
    $project: {
        ratio: {
            $divide: ['$user.followers_count', '$user.friends_count']
        },
        screen_name: '$user.screen_name'
    }
}, {
    $sort: { ratio: -1 }
}, {
    $limit: 1
}])
```
##### 找出发推文@最多的人
```javascript
db.getCollection('tweets').aggregate([{
    $unwind: '$entities.user_mentions'
}, {
    $group: {
        _id: '$user.screen_name',
        count: { $sum: 1 }
    }
}, {
    $sort: { count: -1 }
}, {
    $limit: 1
}])
```

```javascript
db.getCollection('tweets').aggregate([
  { $unwind: '$entities.user_mentions' },
  { 
    $group: {
        _id: '$user.screen_name',
        textSet: { $addToSet: '$text' },
        count: { $sum: 1 }
    }
  },
  {$sort: { count: -1 } },
  { $limit: 1 }
])
```
##### who mention the most unique hashtags
```javascript
db.getCollection('tweets').aggregate([
    {$unwind: '$entities.user_mentions'},
    {$group: {
        _id: '$user.screen_name',
        mset: {
            $addToSet: '$entities.user_mentions.screen_name'
         }
    }},
    {$unwind: '$mset'},
    {$group: {_id: '$_id', count: {$sum: 1}}},
    {$sort: {count: -1}},
    {$limit: 10}
])
```

```javascript
db.getCollection('tweets').aggregate([
    {$unwind: '$entities.user_mentions'},
    {$group: {
        _id: '$user.screen_name',
        mset: {
            $addToSet: '$entities.user_mentions.screen_name'
        }
    }},
    {$project: {count: {$size: '$mset'}, mset: 1}},
    {$sort: {count: -1}},
    {$limit: 10}
])
```

### Index

### 副本集（ReplicaSets）
* 高可用性 （24h 可用）
* 灾难恢复
* 读写分离

### 分片 （Sharding）
* 片键