---
title: GraphQL入门篇
date: 2024-01-29 10:00:00
tags: GraphQL
---

# 背景
传统的RESTful接口开发，大部分情况下都是后台工程师主导的。后台工程师根据当前迭代的需求，设计需要用到的表、以及各表相关的字段，然后和前端工程师商定每个页面上的一些具体的接口字段定义。

这种情况会有如下几个问题：
1. 一个接口可能多端都会用到，不同端用到的接口返回字段集合又不太一样。导致对于每个端，接口的字段利用率不高，数据也不安全。
2. 接口的粒度如果过细，一个页面需要调用的接口数量就会变多，页面加载性能下降。

是否有一种方式，可以让前端可以`按需`获取想要的字段，而后台能够精准返回前端需要字段呢，且无冗余信息？
答案是有的。Facebook 于 2015 年开源了 GraphQL 规范，让前端自己描述自己希望的数据形式，服务端则返回前端所描述的数据结构。


# RESTful接口
假设有一个表，存储所有学生的基本信息。包括学号、姓名、年龄、性别、电话、地址、个人介绍（数据由chatgpt生成，如果雷同，纯属巧合）。
![](https://dsweiblog.oss-cn-shanghai.aliyuncs.com/technote/SCR-20240128-tp3.png)

假如某天需要开发一个学生的联系方式列表，列表条目展示信息有的: 学号、姓名、联系方式。为此接口设计如下：
```js
GET: /students/contactList // 返回一个列表，列表每个元素有id、name、phone三个字段
```
过了几天，需要开发一个学生的家庭住址列表页面，列表条目展示有：学号、姓名、地址。为此后台秉着只返回前端必要信息的原则，设计了新接口:
```js
GET: /students/addressList // 返回一个列表，列表每个元素有id、name、address三个字段
```

又过了几天，需要开发一个学生性别的明细列表，接口如下：
```js
GET: /students/genderList // 返回一个列表，列表每个元素有id、name、gender三个字段
```

以上仅仅是一个学生列表的信息的处理，后台就需要实现三个不同的接口，来应付前端不同需求场景下的数据需求。如果涉及的表和字段更多样，那情况就会变得更复杂，需要设计的接口也会更多（当然另外一种处理方式就是后台不管什么场景，将学生表所有的字段信息都返回。这样是能够降低开发接口的数量，但又会存在每个接口信息冗余的问题）。

# GraphQL是怎么处理的？
想要知道GraphQL是如何解决RESTful面临的问题，可以思考第一段中说的：
> GraphQL让前端自己描述自己希望的数据形式，服务端则返回前端所描述的数据结构。

还是以上一节的例子，返回学生联系方式列表，GraphQL让前端利用如下格式描述自己需要的字段和信息：
```ts
query {
    students {
        id
        name
        phone
    }
}
```
上面的格式中，query表示一个查询请求，查询的内容是students，查询的字段包含id、name、phone三个字段，其他信息不需要返回。

后台返回数据如下
```ts
"students": [
  {
    "id": "1",
    "name": "张三",
    "phone": "13888888888",
  },
  ...
]
```
如果是返回学生地址信息列表，则前端描述如下：
```ts
query {
    students {
        id
        name
        address
    }
}
```
后台返回
```ts
"students": [
  {
    "id": "1",
    "name": "张三",
    "address": "北京市朝阳区建国门外大街18号恒基中心B座2305室\n",
  },
  ...
]
```
可以看到，前端通过控制需要的字段，来让后台返回相应字段的数据。既没有增加接口数量，也没有返回多余的冗余信息，不多不少，蛮美匹配前端要求。完美！

# GraphQL具体工作流程
要实现上一节的基于前端的按需获取字段要求，服务器端需要进行GraphQL改造才能完成。

## GraphQL服务端改造
GraphQL服务端需要完成类型定义、类型操作、类型解析三件事情。

### 类型定义

首先根据学生表结构，定义一个学生类
```ts
  type Student {
    id: ID!
    name: String
    age: Int
    gender: Int
    intro: String
    phone: String
    address: String
  }
```
上面定义的类型叫做GraphQL的对象类型，它是用GraphQL Schema Language来定义的，有点类似于TypeScript，但是和TS稍有不同。

这里的`ID`、`Int`、`String`是GraphQL的标量类型，可以理解为GraphQL的基本类型，其中ID是一个不可重复的字符串，类似于主键，Int和String分别是整形。ID!感叹号表示ID不能为空。

### 类型操作
定义完对象类型，还需要定义一个学生列表的查询操作
```ts
type Query {
    students: [Student]
}
```
Query表示这是一个查询操作，里面定义了一个students的操作类型，它返回的是一个数组，数组中的每个元素是上面定一个的Student类型。

这样，我们完成了GraphQL的类型定义。是不是很简单？

### 类型解析
操作类型定义完，如果没有实现操作类型解析逻辑，那么还是无法相应前端的请求。类型解析器相关的代码类似如下：
```ts
const resolvers = {
    Query: {
        students: async () => {
            const list = await db.query('SELECT * FROM t_students')
            console.log(list)
            return list
        }
    }
}
```
解析器的接口和操作类型的接口基本类似，只不过增加了具体的解析逻辑。

## 服务端完整代码
apollo-server是社区实现的GraphQL服务端库，基于apollo-server，学生列表的服务端的代码如下：
```ts
// db.js
var mysql = require('mysql')
var dbConfig = require('./dbConfig') // 里面的内容需要根据各自本地环境具体配置
var connection = mysql.createConnection({
  host: dbConfig.host,
  user: dbConfig.user,
  password: dbConfig.password,
  database: dbConfig.database
})
module.exports = connection

//////////////////////

// server_graphql.js
const { ApolloServer, gql } = require('apollo-server')
const util = require('node:util');

const db = require('./db')
const port = 3002

db.connect()

// 构造schema
const typeDefs = gql`
  type Query {
    students: [Student]
    student(id: ID!): Student
  }

  type Student {
    id: ID
    name: String
    age: Int
    gender: Int
    intro: String
    phone: String
    address: String
  }
`

db.query = util.promisify(db.query) // 将数据库操作promise化
// 定义resolver
const resolvers = {
  Query: {
    students: async () => {
      const list = await db.query('SELECT * FROM t_students')
      console.log(list)
      return list
    },
  }
}


const server = new ApolloServer({
  typeDefs,
  resolvers,
})

server.listen({port}).then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```
简单解释下上面的代码。我们用到了apollo-server的两个API：gql和ApolloServer。
- 在Apollo Server中，gql 是一个用于定义 GraphQL 模式的标签函数。这个标签函数可以用于创建包含 GraphQL 类型定义的模板字符串。使用 gql 标签函数可以更轻松地在 JavaScript 中书写和组织 GraphQL 模式。
- ApolloServer 函数用来创建和配置 GraphQL 服务器实例的核心构造函数。它接收两个参数，GraphQL模版和Graphql解析器。

当我们运行后，默认会在浏览器端开启一个GUI调试环境。

![](https://dsweiblog.oss-cn-shanghai.aliyuncs.com/technote/graphql-server2.gif)
可以看到，我们在左侧输入我们想要的字段，右侧返回的为字段相应的数据。

## GraphQL客户端
客户端代码其实可以用不同方式实现，甚至可以是命令行。为了方便，可以利用apollo/client来实现一个简单的基于react的客户端。
```ts
import { ApolloClient, InMemoryCache, gql } from "@apollo/client";
import { useEffect, useState } from "react";

function Students() {
  const [students, setStudents] = useState([])
  const allStudents = gql`
    query {
      students {
        id
        name
        address
      }
    }
  `
  const client = new ApolloClient({
    uri: 'http://localhost:3002/',
    cache: new InMemoryCache(),
  })

  useEffect(() => {
    client.query({
      query: allStudents
    }).then(res => {
      setStudents(res.data.students)
    })
  }, [])

  return (
    <>
      <h1>all students</h1>
      <ul>
        {students.map((item) => {
          return <li key={item.id}>{JSON.stringify(item)}</li>
        })}
      </ul>
    </>
  )
}

export default Students

```
至此，一个相对简单且完整的graphQL前后端demo完成了。

# 总结
GraphQL是一种前端的查询语言，方便前端通过定义自己需要的字段，来实现数据的按需加载。本文通过一个简单的数据列表显示的例子，展示了传统Restful接口和GraphQL的差异，同时利用apollo-server和apllo/client，实际展示了GraphQL的前后端相关特性。
需要注意的是，虽然GraphQL在数据的获取和数据利用率上，相比RESTful有明显的提升，但项目改造的成本、后台数据的缓存、数据安全等问题依然需要进一步探讨和深究。
# 代码
本文相关的代码：https://github.com/wdskuki/js-tech-demo/tree/master/graphql