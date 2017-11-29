#MongoDB

###MongoDB的优势
#####灵活的数据模型
[ref link](https://www.mongodb.com/blog/post/mongodb-vs-sql-day-1-2)
#####易扩展
当数据规模大到一个机器装不下了，或者一台机器数据读写负载太高，需要使用cluster的时候，关系型数据库的partitioning, sharding要复杂的手工处理。MongoDB可以自动按照用户给定的sharding key把数据分片，并且动态地平衡各个机器的数据量。
#####大数据量，高性能
NoSQL数据库都具有非常高的读写性能，尤其在大数据量下，同样表现优秀。这得益于它的无关系性，数据库的结构简单。一般MySQL使用Query Cache，每次表的更新Cache就失效，是一种大粒度的Cache，在针对web2.0的交互频繁的应用，Cache性能不高。而NoSQL的Cache是记录级的，是一种细粒度的Cache，所以NoSQL在这个层面上来说就要性能高很多了。

###Get you hands dirty
[Demo data file](https://github.com/ozlerhakan/mongodb-json-files/tree/master/datasets)
#####Import
```bash
mongorestore --drop --host=127.0.0.1 -d twitter-db -c tweets ./tweets.bson
```
#####Export

```bash
mongodump --host=127.0.0.1 -d twitter-db -c tweets -o .
```

###MongoDB basic query
#####点赞数在1000和1100之间的记录
```sql
db.getCollection('tweets').find({'favorite_count': {'$gt': 1000, '$lt': 1100}})
```

#####含Javascript的记录
```sql
db.getCollection('tweets').find({'text': /Javascript/})
```

#####用户Premier League所发的所有记录
```sql
db.getCollection('tweets').find({'user.name': 'Premier League'})
```
#####用户Premier League和BBC News (World)所发的被转发超过500次的记录
```sql
db.getCollection('tweets').find(
{'user.name': {'$in': ['Premier League', 'BBC News (World)']},'retweet_count': {'$gte': 500}},
{'_id': 0, 'text': 1, 'user.name': 1})
```
#####提到Allan Yeung的记录
```sql
db.getCollection('tweets').find({
'entities.user_mentions.name': 'Allan Yeung'
})
```

###Aggregation Framework
#####按发tweet记录数对用户排序
```sql
db.getCollection('tweets').aggregate([
{'$group': {'_id': '$user.screen_name', 'count': {'$sum': 1}}},
{'$sort': {'count': -1}}
])
```
#####找出粉丝人数与朋友人数比例最大的用户
```sql
db.getCollection('tweets').aggregate([
{'$match': {'user.friends_count': {'$gt': 0}, 'user.followers_count': {'$gt': 0}}},
{'$project': {'ratio': {'$divide': ['$user.followers_count', '$user.friends_count']}, 'screen_name': '$user.screen_name'}},
{'$sort': {'ratio': -1}},
{'$limit': 1}
])
```
#####找出发推文@最多的人
```sql
db.getCollection('tweets').aggregate([
    {'$unwind': '$entities.user_mentions'},
    {'$group': {'_id': '$user.screen_name', 'count': {'$sum': 1}}},
    {'$sort': {'count': -1}},
    {'$limit': 1}
])
```

```sql
db.getCollection('tweets').aggregate([
    {'$unwind': '$entities.user_mentions'},
    {'$group': {'_id': '$user.screen_name', 'textSet': {'$addToSet': '$text'}, 'count': {'$sum': 1}}},
    {'$sort': {'count': -1}},
    {'$limit': 1}
])
```
#####who mention the most unique hashtags
```sql
db.getCollection('tweets').aggregate([
    {'$unwind': '$entities.user_mentions'},
    {'$group': {
        '_id': '$user.screen_name',
        'mset': {
                '$addToSet': '$entities.user_mentions.screen_name'
         }
    }},
    {'$unwind': '$mset'},
    {'$group': {'_id': '$_id', 'count': {'$sum': 1}}},
    {'$sort': {'count': -1}},
    {'$limit': 10}
])
```

```sql
db.getCollection('tweets').aggregate([
    {$unwind: '$entities.user_mentions'},
    {$group: {
        _id: '$user.screen_name',
        mset: {
            '$addToSet': '$entities.user_mentions.screen_name'
        }
    }},
    {$project: {count: {$size: '$mset'}, mset: 1}},
    {$sort: {count: -1}},
    {$limit: 10}
])
```

###Index
