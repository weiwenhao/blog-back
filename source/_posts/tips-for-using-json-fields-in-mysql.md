---
title: MySQL 中 JSON 字段的使用技巧
date: 2018-12-12 11:13:40
tags: mysql json
---

mysql5.7.8之后开始原生支持json. 在类似mongodb这种nosql数据库中,json存储数据是非常自然的, 在mysql中合理的使用json,能够带来极大的便利


<!-- more -->


#### Json字段的使用场景

在读laravel手册举例子时,我们经常会看到 `$user->is_admin` 来判断用户是否为管理员,但是在用户表中,admin往往只占很小一部分.如果单开一个is_admin字段是很没有必要的行为.数据库中会有大量的无意义数据存储, 我们可以为user表创建一个 json 字段,来存储我们的is_admin字段

```json
[
    {
        id: 1,
        username: 'weiwenhao',
        rest: { // 冗余字段
            is_admin: 1
        }
    },
    {
        id: 2,
        username: 'eienao',
        rest: null
    }
]
```

> 当然即使不使用json,我们也不会使用is_admin来判断是否为管理员.
>
> 可以通过新增admin表或者RABC来标志管理员



依旧是用户表, 很常见的一个需求是第三方登录,如果我们使直接在user表新增`facebook_id,facebook_email,facebook_phone_number,google_id,....`字段, 可以预见这会造成大量的无意义数据(即使他们不占用内存,或者影响性能)

> 一种解决办法是 使用一对多关系来解决, 既建立一个 第三方登录表来存储第三方登录的id/email/phone_number等

但是我更喜欢使用json字段来解决这个问题

```
[
   {
        id: 1,
        username: 'weiwenhao',
        rest: {
            is_admin: 1,
            facebook_id: 2348234,
            facebook_phone_number: 2834723234,
        }
    },
    {
        id: 2,
        username: 'eienao',
        rest: {
            google_id: 2348234,
            google_email: xxx@gmail.com
        }
    }
]
```

可以看出,使用json字段使数据表的设计更加自然,集中,业务也相应的更加的简单方便.



#### Json字段在laravel中的使用

首先是迁移文件 `$table->json('rest')->nullable();` 

laravel对json的使用进行了一定的优化,对于更新和创建我们可以.

```php
$user = new User;
$user->{'rest->google_id'} = 'xxx'; 
# 如果你的rest字段为null,那么上面的操作会使 null 会变成 {google_id: "xxx"}, 不需要再做 是否为null的判定啦
# 如果仅使用上面的插入操作,也不需要在使用模型的修改器来吧 json => array, array => json啦
```

> 当rest字段的值为null时,批量操作无法执行, 类似  `update(['rest->google_id' => 'xxx'])` 这样的操作执行无效,因此更推荐上面的方式来进行更新操作

对于查找操作可以方便的使用

`User::where('rest->google_id','xxx')->firstOrFail()`

>  关于检索的效率问题,在后面内容中给出解决方案



#### Generated Column (生成列)

5.7新增了生成列, 生成列的值是根据列定义中包含的表达式计算得来.官方示例:计算直角三角形的斜边的长度

```sql
CREATE TABLE triangle (
  sidea DOUBLE,
  sideb DOUBLE,
  sidec DOUBLE AS (SQRT(sidea * sidea + sideb * sideb)) # AS (expression) 为生成列的核心语法
);
INSERT INTO triangle (sidea, sideb) VALUES(1,1),(3,4),(6,8);

# 对于上面的插入,查询可以得到如下结果

mysql> SELECT * FROM triangle;
+-------+-------+--------------------+
| sidea | sideb | sidec              |
+-------+-------+--------------------+
|     1 |     1 | 1.4142135623730951 |
|     3 |     4 |                  5 |
|     6 |     8 |                 10 |
+-------+-------+--------------------+
```

上面的 sidec的值 是根据sidea和sideb计算得来, 并未实际的存储在磁盘中.mysql5.7之前我们想要实现上面的需求可能会这样写sql语句

```sql
SELECT *,(SQRT(sidea * sidea + sideb * sideb)) as sidec FROM triangle;
```

上面既生成列的主要作用, 实际上生成列有两种子类型,上面的例子属于 virtual (虚拟) 类型的生成列, 其并没有将sidec的值实际存储在磁盘中.

除了virtual, 生成列还支持 stored类型,其创建语句为

```sql
#...
sidec DOUBLE AS (SQRT(sidea * sidea + sideb * sideb)) STORED # stored不指定则默认为 virtual
#...
```

当行创建或者更新时, 会重新计算 sidec并将其存储在磁盘中

生成列的另一个重要的特性是可以根据**生成列表达式的计算结果建立索引**. 其建立索引的方式和普通字段创建索引的方式一致.1

```sql
CREATE TABLE triangle (
  sidea DOUBLE,
  sideb DOUBLE,
  sidec DOUBLE AS (SQRT(sidea * sidea + sideb * sideb)) # AS (expression) 为生成列的核心语法
  INDEX(`sidec`)
);
```

索引本身也是存储在磁盘中的实际存在的物质, 因此 virtual 生成列 + 索引,可以达到存储空间的最有效利用. 

> 对于stored 生成列 + 索引, 通常不会访问到存储在磁盘中stored 生成列,而是直接访问索引.因此没有必要使用stored生成列



#### 使用生成列为json中的字段添加索引

已user表的rest.google_id为例,建表操作

```sql
#...
`rest` json NULL,

# JSON_EXTRACT(`rest`,'$.google_id') 等价于 `rest`->'$.google_id'
# 5.7.13版本后 
# JSON_UNQUOTE(JSON_EXTRACT(`rest`,'$.google_id')) 等价于 `rest`->>'$.google_id'
# 使用生成列为json添加索引时,请务必使用 JSON_UNQUOTE(JSON_EXTRACT(`rest`,'$.google_id'))/->>
`google_id` varchar GENERATED ALWAYS AS (`rest`->>'$.google_id')) NULL

UNIQUE INDEX(`google_id`)
#...
```

在laravel迁移文件中

```php
$table->json('rest')->nullable();

$table->string('rest')->nullable()->unique()->virtualAs('`oauth`->>"$.google_id"');
```

有了索引后,当我们执行查询操作

```sql
select * from users where `rest`->'$.google_id' = 'xxx' # 通常使用这种更加简单的形式
select * from users where `rest`->>'$.google_id' = 'xxx'

# 上面两种表达式会被mysql的优化器在查询阶段自动优化为 select * from users where google_id = 'xxx'
```



#### 一些补充

- `virtualAs(oauth->"$.google_id"');` 使用 **->**符号来创建生成列会出现无法使用索引的情况, 原因不是很明了,需要继续研究一下手册. 另外对于创建语句 `GENERATED ALWAYS`的作用也不是很明了.

- 关于null, 经常会看到一种言论是mysql中使用null作为字段默认值会出现无法索引的情况.但经过查询了解,发现这是一种老中医理论. 我更倾向于使用null作为默认值, 而不是 ''/0/0.0 ,我认为null的表达性更好, laravel中也无时无刻不在提现这种思想.
- 关于json的使用, 最近的项目中,我大部分核心表都有一个json字段,做一些非核心数据的存储和冗余. 

