### **关于mongdb教程的使用**

1. 创建数据库: use dbnane;

    
    use test

这个时候数据库不会显示，只有db.test.insert()以后才会有该数据库的数据

2. 插入数据 db.tableName.insert()

        
    db.test.insert({
            'title':'mongo',
            'desc': 'mongo is a kind of no sql db',
            'by': 'asheng',
            'url': 'www.baidu.com',
            'tags': ['mongo', 'nosql'],
            'likes': 100
            })


insert 和 save都会插入数据，但是如果save传入了id的话，将会是更新操作


3. 更新操作


       db.tableName.update(<query>,
       <update>,
       {
       upsert:boolean,
       multi:boolean
       writeConcern:document
       })

- query是update的条件，类似于sql中的where
- update是更新操作，如$set、$incr等，类似sql中的update
- upsert可选，如果不存在则插入，true的话表示开启
- multi可选，默认是false，只更新找到的第一条，true则更新所有匹配的记录
- writeConcern可选，跑出异常的级别

    
    db.test.update({'title': 'mongo'}, {$set:{'title': 'MONGO'}}, {multi:true})

如果save操作的id存在，则也是更新操作

    db.test.save({
    "_id" : ObjectId("5d4d17dd4242720801b3c202"),
    "title" : "hello mongo",
    "desc" : "mongo is a kind of no sql db",
    "by" : "asheng",
    "url" : "www.baidu.com",
    "tags" : [ 
        "mongo", 
        "nosql"
    ],
    "likes" : 100.0
    })

save的用法

    db.tableName.save(<document>, {writeConcern: <document>})


4. 删除数据操作


    db.tableNames.remove(<query>, {jsutOne: <boolean>, writeConcern: <document>})

query：可选，删除文档的条件
justOne: 如果设置成true，则只删除一个文档，默认false:删除所有匹配条件满足的文档
writeConcern: 可选，跑出异常的级别

    db.test.remove({'by': "asheng"})
    
    db.test.remove({}) 删除所有数据


5. 查询操作


    db.tableName.find(query, projection)

query: 可选，适应查询操作符指定查询条件
projection: 可选，使用投影操作符制定返回的健，查询时返回文档中所有的健只需要忽略该参数即可

findOne()只返回一个文档



操作 | 格式 | MongDB | RDBMS
---|---|---|---
等于 | {key: value} | db.test.find({'by': 'asheng'}) | where by='asheng'
小于 | {key: {$ls: value}} | db.test.find({'likes': {$lt:50}}) | where likes < 50
小于或等于 | {key: {$lte: val}} | db.test.find({'likes': {$lte: 50}}} | where likes <= 50
大于 | {key: {$gt: val}} | db.test.find({'likes': {$gt: 50}}) | where likes > 50
大于或等于 | {key: {$gte: val}} | db.test.find({'likes': {$gte: 50}}) |  where likes >= 50
不等于 | {key: {$ne: val}} | db.test.find({'likes': {$ne: 50}}) | where likes != 50

MongDB的find可以传入多个key，每个key已,隔开，即SQL的AND操作


    db.test.find({'likes': 100}, {'by': 'asheng'}, {'title':{$ne: 'hello'}})


MongDB中的OR条件，使用$or关键字


    db.test.find({$or: [{key: value}, {key: value}]})
    
    
    db.test.find({'likes': 100}, {'by': 'asheng'}, {'title':{$ne: 'hello'}}, {$or:[{'likes': {$eq:100}}]})



范围查找，类似于between and

    db.test.find({'likes': {$gte: 100, $lt: 200}})


5. $type操作符号，给予BSON类型检索集合中匹配的数据类型并返回


    db.test.find({'title': {$type: 2}}) 其中2代表string类型


6. 如果获取多少行数据，使用Limit方法

   
    db.tableName.find().limit(Number)
    db.test.find({'title': {$type: 2}}).limit(2)

7. 如果从第多上行获取，使用Skip方法，表示跳过第记录数


    db.test.find({'title': {$type: 2}}).skip(2).limit(2)

8. 排序，使用sort方法对数据进行排序，指定字段以及使用1 和 -1来指定排序方式 1是生序 2是降序

   
    db.tableName.find().sort({key: 1})
    db.test.find().sort({'likes': 1})

9. 创建索引 createIndex()方法

   
    db.tableName.createIndex(keys, options)

其中key是创建索引对字段，1是指按照升序创建索引  -1是按照降序创建索引

    db.test.createIndex({'likes': 1}})
    db.test.createIndex({'likes': 1, 'title': 1})
    
    
    db.tableName.createIndex({key, option}, {backgroud: true, unique: true, name:indexName, sparse:true, weights: 1-9999, default_language:es})


其中后面对参数的意义如下:

- backagroud: 以后台方式创建索引

- unique:是否创建唯一索引

- name: 索引名

- sparse: 对不存在的字段不启用索引，默认是false。如果设置成为true, 在索引字段红不会查询出不包含对应字段的文档

- weights: 索引权重值，表示该索引相对其他索引的得分权重

- default_language: 对文本索引，该参数决定了停用词以及词干和词器的规则的列表，默认英语


10. mongodb聚合，主要是处理数据，如平均值、求和等


    db.test.aggregate([{
    $group: {_id: '$by', num: {$sum : 1}}
    }])

类似于 select by, count(*) from test group by 'by';


表达式 | 描述 | MongoDB
---|---|---
$sum | 求和 | db.test.aggregate([{$group: {_id: '$by', num:{$sum: 1}}}])
$avg | 求平均值 | db.test.aggregate([{$group: {'_id','$by', avg: {$avg: '$likes'}}}])
$min | 最小值 | db.test.aggregate([{$group: {'_id': '$by', min: {$min: '$likes'}}}])
$max |最大值 | 替换min为max
$push | 在结果文档中插入值到一个数组中 | db.test.aggregate([{$group: {_id: '$by', url_array: {$push: '$url'}}}]) **注释: 就是将同一组到数据到某个字段放到url_array中 **
$addToSet | 在结果文档中插入值到一个数组中，但不创建副本 | db.test.aggregate([{$group: {$id: '$by', url_set: {$addToSet: "$url"}}}])
$first | 根据资源文档但排序获取第一个文档数据 | db.test.aggregate([{$group: {$id: "$by", first_url: {$first: '$url'}}}])
$last | 根据资源文档获取最后一个文档数据 | 同上，替换first为last


11. 管道的概念

管道就是将MongoDB文档在一个管道处理完毕后将结果传递给下一个管道处理，管道操作可以是重复的，表达式：处理输入文档并输出，表达式是无状态的，只用于计算当前管道的文档面部处理其他文档

- $project: 修改输入文档，可以用来重命名，增加或删除域，也可以用于创建计算结果以及嵌套文档

- $match: 用于过滤数据，只输出符合条件的文档

- $limit: 限制返回文档树

- $unwind: 将文档中的某一个数组类型字段分成多条，每条包含数组中的一个值

- $group: 将集合中的文档分组，用于统计结果

- $sort: 将输入文档排序后输出

- $geoNear: 输出接近某一地理位置的有序文档


    db.article.aggregate({$project: {title: 1, auth:1}})

返回的结果就只有titile, auth和_id三个字段，如果不想输出id。可以_id:0就可以把_id字段屏蔽


    db.article.aggregate([{$match: {score: {$gt: 10, $lte: 90}}}, {$group: {_id: null, count: {$sun: 1}}}])



12. 分片集搭建


    /mongodb4/bin/mongod --port 27017 --dbpath "/Users/asheng/DevTools/mongodb/data-repl1" --logpath "/Users/asheng/DevTools/mongodb/logs-repl1/mongod.log" --replSet rs0
    
    /mongodb4/bin/mongod --port 27018 --dbpath "/Users/asheng/DevTools/mongodb/data-repl2" --logpath "/Users/asheng/DevTools/mongodb/logs-rep2/mongod.log" --replSet rs0
    
    /mongodb4/bin/mongod --port 27019 --dbpath "/Users/asheng/DevTools/mongodb/data-repl3" --logpath "/Users/asheng/DevTools/mongodb/logs-repl3/mongod.log" --replSet rs0

    执行命令
    rs.initiate()
    rs.add("localhost:7018")
    rs.add("localhost:7018")

13. 集群搭建

搭建集群后，会按照某个键值进行分片，分片到不通的数据库中，这样的话，保证数据库的负载差不多

搭建shard集群

    ./mongodb4/bin/mongod --port 27017 --dbpath "/Users/asheng/DevTools/mongodb/data-repl1" --logpath "/Users/asheng/DevTools/mongodb/logs-repl1/mongod.log" --replSet rs0 -shardsvr
    
    ./mongodb4/bin/mongod --port 27019 --dbpath "/Users/asheng/DevTools/mongodb/data-repl3" --logpath "/Users/asheng/DevTools/mongodb/logs-repl3/mongod.log" --replSet rs0 -shardsvr
    
    ./mongodb4/bin/mongod --port 27018 --dbpath "/Users/asheng/DevTools/mongodb/data-repl2" --logpath "/Users/asheng/DevTools/mongodb/logs-repl2/mongod.log" --replSet rs0 -shardsvr
    
    执行命令
    rs.initiate()
    rs.add("localhost:7018")
    rs.add("localhost:7018")

搭建configserver集群

    ./mongodb4/bin/mongod --port 27020 --dbpath "/Users/asheng/DevTools/mongodb/data-config" --logpath "/Users/asheng/DevTools/mongodb/logs-config/mongod.log"  --replSet rs-config --configsvr
    
    必须以集群方式搭建
    执行命令
    rs.initiate()

搭建mongos

    ./mongodb4/bin/mongos --port 27021 --configdb rs-config/127.0.0.1:27020 --logpath "/Users/asheng/DevTools/mongodb/logs-route/mongod.log"
    
    执行如下命令
    /usr/local/mongoDB/bin/mongo admin --port 40000
    
    添加分片，即数据落到如下分片后可以做多重备份，以及主备切换
    db.runCommand({ addshard:"rs0/localhost:27020,localhost:27020,localhost:27020"})
    db.runCommand({ addshard:"rs1/localhost:27020,localhost:27020,localhost:27020"})
    
    对那些数据库开启切片
    db.runCommand({ enablesharding:"test" }) #设置分片存储的数据库
    
    对数据表进行切片，以及切片的key
    db.runCommand({ shardcollection: "test.log", key: { id:1,time:1}})
    
