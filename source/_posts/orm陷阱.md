---
title: ORM 陷阱
date: 2019-03-01 17:11:09
tags:
---

产品实现中有一类常见的需求是,取出一组数据, 且这组数据中的每一项都携带固定数量的关联数据.



如取出一组热门作者及他们最近发表的3篇文章

![](https://cdn.learnku.com/uploads/images/201903/01/10960/7pD3giMAlh.png!large)

但在MySQL(ORM)中这种需求并不能比较完美的实现.

你看到这种需求的第一眼可能会想到这么写

> users: id
>
> posts: id,user_id
>
> user 通过外键**user_id** **一对多**关联 post

```php
$users = \App\Models\User::with(['posts' => function ($query) {
    $query->limit(3);
}])->limit(10)->get();
```

但这并不能满足该需求,并且**这是一种错误的写法.** 这里的期望是10个作者,每个作者取出3篇帖子,一共取出了30篇帖子,实际生成的sql语句是

```sql
select
  *
from
  `posts`
where
  `posts`.`user_id` in (1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
limit
  3
```

你应该一眼就看出了不可行的原因.虽然知道了这样不可行,但是仔细去想却发现,自己也想不出不出满足这个需求的sql语句,mysql并没有类似each_limit的语法.

> sql大佬请忽略我这句话!!

问题与陷阱应该阐述的比较清楚了,接下来看看几个可行的解决方案



## PLAN A

使用 N + 1的sql查询方案.  

```php
$users = \App\Models\User::limit(10)->get();

$users = $users->map(function ($user) {
    //可以考虑$user->id缓存,在保证了速度的同时避免大面积的缓存重建
    $user->posts = $user->posts()->limit(3)->get();
    
    return $user;
});

return $users;
```

这种做法的一个好处是思路足够简单直白,没有复杂sql.在缓存的加持下可以避免N+1问题.



## PLAN B

对PLAN A稍微修改一下就构成PLAN B, 既UNION ALL解决方案. 通过mysql的UNION ALL将上面需要进行的10次查询联合成一次查询.

```php
$users = \App\Models\User::limit(10)->get();

// 拼接 union all
$posts = $users->map(function ($user) {

    return $user->posts()->limit(3);

})->reduce(function ($carry, $query) {

    return $carry ? $carry->unionAll($query) : $query;

})->get();

// 将posts按照一对多的关系relation到users中
$relation = \App\Models\User::query()->getRelation('posts');
$relation->match(
    $relation->initRelation($users->all(), 'posts'),
    $posts, 'posts'
);

return $users;
```

sql如下

```sql
(
  select
    *
  from
    `posts`
  where
    `user_id` = 355
  limit
    3
)
union all
  (
    select
      *
    from
      `posts`
    where
      `user_id` = 234
    limit
      3
  )
union all
  (
    select
      *
    from
      `posts`
    where
      `user_id` = 232
    limit
      3
  )
...
```

上面的查询有效的避免了PHP与MySQL之间的I/O耗时,并且UNION ALL可以有效的利用索引.

在50w条posts数据时,查询平均耗时 0.002s,算是不错的表现了. Explain如下

![](https://cdn.learnku.com/uploads/images/201903/01/10960/Vk4RefITFr.png!large)



## PLAN C

合理利用MySQL中的变量语法我们也可以实现这个需求,只需要用一个变量帮我们编码一下即可

```sql
SELECT
	posts.*,
	@number := IF (@current_user_id = `user_id`, @number + 1, 1) AS number,
	@current_user_id := `user_id`
FROM
	(select * from `posts` where `posts`.`user_id` IN (572, 822, 911, 103, 234, 11, 999, 333, 121, 122) order by `posts`.`user_id` ASC) AS posts
HAVING
	number <= 3
```

简单解析一下这个sql语句.  

FORM 为一个子查询,初步筛选出我们需要的作者的所有文章, 且**正序排列**后生成一个临时表.

SELECT 为上面临时表添加标号,添加的方式如下. (你需要从上往下一行行一行的观察,与select的执行方式一致即可)

> MySQL中调用未定义的变量,其值默认为null.

|  id  | user_id | @current_user_id |           if判断            | @number |
| :--: | :-----: | :--------------: | :-------------------------: | :-----: |
|  1   |    1    |       null       |   false, number被赋值为1    |    1    |
|  2   |    1    |        1         | true, @number = @number + 1 |    2    |
|  3   |    1    |        1         | true, @number = @number + 1 |    3    |
|  4   |    1    |        1         | true, @number = @number + 1 |    4    |
|  5   |    2    |        1         |   false, number被赋值为1    |    1    |
|  6   |    2    |        2         | true, @number = @number + 1 |    2    |
|  ..  |   ..    |        ..        |             ..              |   ..    |

HAVING执行于SELECT之后,其再次筛选上面的临时表,只取number <= 3的 行.

在执行效率方面,50万posts数据时,查询平均耗时在 0.112s,不算太差,也不算太好的一个表现. Explain如下

![](https://cdn.learnku.com/uploads/images/201903/01/10960/ckcVQC5M4X.png!large)

通过sql语句我们也可以预见. **from子查询的结果集的数量,会直接影响后面标号的效率.** 在子查询的结果集不多时,该查询会有更好的表现.

PLAN C有一个额外的好处是,其有不错的laravel扩展支持,能够在不影响原有ORM操作的情况下实现该需求

> [staudenmeir/eloquent-eager-limit](https://github.com/staudenmeir/eloquent-eager-limit)



## PLAN D

我们也可以添加一张中间表来维持这种关联关系,比如作者与其最新发表的文章部分.我们可以创建一张

`user_latest_post:user_id,post_id`,  然后在User模型中添加一条多对多的关联关系

```php
# User.php

public function latestPosts()
{
    return $this->belongsToMany(Post::class, 'user_latest_post')
}
```

使用

```php
$users = \App\Models\User::with(['latestPosts'])->limit(10)->get();
```

该方案更加的符合ORM的关联关系模型,查询的效率上也较高. 但是需要增加对`user_latest_post`表的维护成本

## #

总结来说上面四种方案都还算不错,每种都有自己的优势,可以根据自己的情况选择合适的方案. 当然如果你有其他的解决方案,欢迎留言~