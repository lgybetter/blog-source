---
title: MongoDB原生命令CURD
date: 2017-03-22 22:39:05
tags: 
  - MongoDB
categories:
  - 数据库
---
> 《MongoDB权威指南》学习摘录

## Insert 

### 使用insert方法

- 该操作文档会自动生成一个"_id"键

```bash
>db.foo.insert({“bar” : “baz”})
```

### 批量插入，使用batchInsert函数

```bash
>db.foo.batchInsert([{ "_id" : 0 } , { "_id" : 1 } , { "_id" : 2 }])
```

当前mongodb能接受的最大消息长度是48MB,并且如果在执行批量插入过程中有一个文档插入失败，则在该文档之前的所有文档都会成功插入到集合中，而这个文档之后的所有文档全部插入失败。


## Remove

### 使用remove方法

```bash
//删除foo集合的所有文档
>db.foo.remove()

//删除指定查询文档作为可选参数
>db.mailing.list.remove( { “opt-out” : true } )
```

### 使用drop()比直接删除集合会更快

```bash
>db.foo.drop()
```

## Update

### 文档替换

```json
{
  "_id": ObjectId("..."),
  "name": "joe",
  "friends": 32,
  "enemies": 2
}
```

现在需要更新"friends"和"enemies"两个字段到"relationships"子文档中

```bash

>var joe = db.users.findOne({"name":joe});
>joe.relationships = {"friends":joe.friends,"enemies":joe.enemies};
{
  "friends":32,
  "enemies":2
}
>joe.username = joe.name;
"joe"
>delete joe.friends;
true
>delete joe.enemies;
true
>delete joe.name;
true
>db.users.update({"name":"joe"},joe);
```

更新后的文档

```json
{
  "_id":ObjectId("..."),
  "username":"joe",
  "relationships":{
    "friends":32,
    "enemies":2
  }
}
```

常见的错误是查询条件匹配到多个文档，然后更新的时候回由于第二个参数的存在就产生重复的"_id"值,对于这种情况要使用"_id"值来进行查询


### 使用修改器

- "$inc"修改器使用

当有人访问页面的时候，就通过url找到该页面，然后使用"$inc"修改器增加"pagerviews"的值

```json
{
  "_id":ObjectId("..."),
  "url":"www.lgybetter.com",
  "pagerviews":52
}
```

```bash
>db.analytics.update({"url":"www.lgybetter.com"},{"$inc" : {"pagerviews" : 1}})
```

注意，使用修改器时，"_id"并不会改变，而文档替换过则会

- "$set"修改器使用

```bash
>db.user.update({"_id":ObjectId(...)},{"$set" : {"favorite book" : "War and Peace"}})
```

于是更新后的文档就有了"favorite book"键，如果要继续修改则：

```bash
>db.user.update({"_id":ObjectId(...)},{"$set" : {"favorite book" : "Green Eggs and Ham"}})
```
也可以变为数组的类型

- 使用"$unset"可以将这个键完全删除：

```bash
>db.user.update({"name":joe},{"$unset" : {"favorite book" : 1}})
```

- "$push"修改器使用
	
如果数组存在，"$push"会向已有的数组的末尾加入一个元素，如果没有改数组就创建一个新的数组

```json
{
  "_id":ObjectId("..."),
  "title":"A blog post",
  "content":"..."
}
```

```bash
>db.blogs.posts.update({"title" : "A blog post"},{"$push" : {
	"comments" : {"name" : "joe","email" : "joe@example.com","content":"nice post."}
}})
>db.blog.posts.findOne()
{
  "_id":ObjectId("..."),
  "title":"A blog post",
  "content":"...",
  "comments": [
    {
      "name":"joe",
      "email":"joe@example.com",
      "content":"nice post"
    }
  ]
}
```
注意，"$push"会创建一个数组的类型，可用于动态添加数据


- 用"$push"一次添加多个数据，配合使用"$each"

```bash
>db.stock.ticker.update({"_id":"GOOG"},{
  "$push" : {"hourly" : { "$each" : [
    562.776,562.790,559.123
  ]}}
})
```

- 使用"$slice"来固定数组的最大长度

```bash
>db.movies.find({"genre" : "horror"},{
  "$push" : { "top10" : {
      "$each" : ["Nightmare on Elm Street" , "Saw"],
      "$slice" : -10,
      "$sort" : {"rating" : -1}
    }
  }
})
```

注意，"$slice"的值应该是负整数,"$sort"可以用来对数组中的所有对象进行排序，不能只将"$slice"或者"$sort"与"$push"配合使用，且必须使用"$each"。

- 使用"$ne",使得数组变为数据集合

```bash
>db.paper.update({"authors cited" : {"$ne" : "Richie"},
  {"$push" : {"authors cited" : "Richie"}}
})
```

- "$addToSet"和"$each"组合使用，效果是与"$ne"一样，但是可以用来同时插入多条数据

```bash
>db.users.update({"_id" : ObjectId("...")}, {"$addToSet" :
  {"emails" : {"$each" :
    ["joe@php.net","joe@example.com","joe@python.org"]
  }}
})
```

- 使用"$pop"删除数组元素{"$pop" : {"key" : 1}}从数组末尾删除，{"$pop" : {"key" : -1}}从数组头部删除

- 使用"$pull"删除元素

```bash
>db.lists.update({},{"$pull" : {"todo" : "laundry"}})
```

### 使用upsert

upsert是一种特殊的更新，要是没有找到符合更新条件的文档，就会以这个条件和更新
文档为基础创建一个新的文档。

```bash
>db.analytics.update({"url" : "/blog"} , {"$inc" : {"pageviews" : 1}} ,true)
```

- 有时候，需要在创建文档的同时创建字段并为它赋值，但是在之后的所有更新操作中，该字段的值就不再改变。这时候就使用"$setOnInsert"

```bash
>db.users.update({} , {"$setOnInsert" : {"createdAt" new Date()}},true)
```

### 更新多个文档

具体的操作值通过:

```bash
>db.runCommand({getLastError:1}) 
```

返回结果

```json
{
  "err":null,
  "updatedExisting":true,
  "n":5,
  "ok":true
}	
```

## Find

### find入门操作

- 查询该集合下的所有文档
```bash
>db.c.find()
```

- 查询特定值的文档,可以同时添加多个条件
```bash
>db.users.find({"age" : 20})
>db.users.find({"age" : 20 , "name" : "lgy"})
//指定需要返回的键
>db.users.find({} , {"username" : 1,"email" : 1})
//返回结果
{
  "_id":ObjectId("..."),
  "username":"lgy",
  "email":"lgy@example.com"
}
```

- 通过查询，默认情况下回返回"_id"这个键，如果希望不出现其他的键值，就使用如下：
```bash
>db.users.find({},{"username" : 1,"_id" : 0})
//返回结果
{
	"username":"lgy"
}
```

### 查询条件

- 查询条件：

"$lt","$lte","$gt","$gte"就是全部的比较操作符，分别对应<,<=,>,>=

```bash
//查询18 ~ 30岁的用户：
>db.users.find({"age" : {"$gte" : 18 , "$lte" : 30}})
```

"$ne"表示不等于某个特定的值
```bash
//查询用户名不为"lgy"的所有用户
>db.users.find({"username" : {"$ne" : "lgy"}})
```

- OR查询：

"$in" 和 "$or"两种方式进行OR查询

```bash
>db.raffle.find({"ticket_no" : {"$in" : [725,542,390]}})
```

"$nin"是与"$in"相反的

```bash
>db.raffle.find({"$or" : [{"ticket_no" : 725},{"winner" : true}]})
```

- 两者结合使用：
```bash
>db.raffle.find({"$or" : [{"ticket_no" : {"$in" : [725,542,300]}},{"winner" : true}]})
```

$not

  "$not"是元条件句，即可以在任何其他条件之上。

	"$mod"通过传入两个参数，第一个用来作为除数，第二个是用来判定余数是否为此数字

```bash	
>db.user.find({"id_num" : {"$not" : {"$mod" : [5,1]}}})
```


### 特定类型的查询

- null

null不仅会匹配某个键的值为null的文档，而且还匹配不包含这个键的文档

```bash
>db.c.find({"z" : null})
```

如果仅仅想匹配键值为null的文档，则可以加"$exists"条件：

```bash
>db.c.find({"z" : {"$in" : [null], "$exists" : true}})
```

- 正则表达式

```bash
>db.users.find({"name" : /joey?/i})
```

- 查询数组

$all,可以用来进行多个元素匹配数组

```bash	
//这样查找就可以匹配到同时存在的两个元素的文档
>db.food.find({"fruit" : {"$all" : ["people","banana"]}})
```

$size用来查询特定长度的数组

```bash
>db.food.find({"furit" : {"$size" : 3}})
```

$slice用来返回某个匹配数组元素的一个子集

```bash
//返回前十条评论，如果把10换成-10就是返回后十条评论
>db.blog.posts.findOne(criteria,{"comments" : {"$slice" : 10}})

//这个操作会跳过前面23个元素，返回第24~33个元素
>db.blog.posts.findOne(criteria,{"comments" : {"$slice" : [23,10]}})
```
返回一个匹配的数组元素

```bash
>db.blog.find({"comments.name" : "bob"}, {"comments.$" : 1})
//返回结果
{
  "id" :　ObjectId("..."),
  "comments" : [
    {
      "name" : "bob",
      "email" : "bob@example.com",
      "content" : "good post"
    }
  ]
}
//这样就只会返回第一条评论
```

- 查询内嵌文档

现在有这样的数据

```json
{
  "name": {
    "first":"joe",
    "last":"Schmoe"
  },
  "age":45
}
```

对内嵌文档的查询

```bash
//用点表示法表达"进入内嵌文档内部"的意思
>db.people.find({"name.first" : "joe" ,"name.last" : "Schmoe"})
```

要正确指定一组条件，而不必指定每个键，就需要使用"$elematch"

```bash
>db.blog.find({"comments" : {$elematch" : {"author" : "joe", "score" : {"$gte" : 5}}}})
```

### 游标

- 游标查询的具体操作：

```bash
>for(i = 0; i <100; i ++) {
  db.collection.insert({x : i});
}
```
