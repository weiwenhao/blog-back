---
title: '编写具有描述性的 RESTful API (一): Workflow'
date: 2019-03-12 17:11:09
tags:
---

这是个系列,将会以结构较为简单明了的 [简书](https://www.jianshu.com/) 为参考,编写一套简洁,**可读**的社区类型的 **RESTful** API.

> 我使用的laravel版本是5.7, 且使用 [tree-ql](https://github.com/weiwenhao/tree-ql) 作为api开发基础工具.
>
> 该系列并不会一步一步来教你怎么实现,只会阐明一些基本点,以及一些关键的地方
>
> 相关的代码我会放在 [weiwenhao/community-api](https://github.com/weiwenhao/community-api) 你可以随时参阅一些细节部分



## 开始咯

#### [建表](https://learnku.com/docs/laravel/5.7/migrations/2291)

先来看看**[设计稿](https://www.jianshu.com/p/aff99d1d194b)**,根据设计稿可以设计出基础的帖子表结构. 当然这不会是最终的表结构,后面会根据实际情况来一点点完善该表.

```php
Schema::create('posts', function (Blueprint $table) {
    $table->increments('id');
    $table->string('code')->index();
    $table->string('title');
    $table->string('description');
    $table->text('content')->nullable();
    $table->string('cover')->nullable();
    $table->unsignedInteger('comment_count')->default(0)->comment('评论数量');
    $table->unsignedInteger('like_count')->default(0)->comment('点赞数量');
    $table->unsignedInteger('read_count')->default(0)->comment('阅读数量');
    $table->unsignedInteger('word_count')->default(0)->comment('字数');
    $table->unsignedInteger('give_count')->default(0)->comment('赞赏数量');

    $table->unsignedInteger('user_id')->index();

    $table->timestamp('published_at')->nullable()->comment('发布时间');
    $table->timestamp('selected_at')->nullable()->comment('是否精选/精选时间');
	$table->timestamp('edited_at')->nullable()->comment('内容编辑时间');
    
    $table->timestamps();
});
```

然后看看评论表.简书的评论并不是无限级的,而是分为两层,结构简单.

```php
Schema::create('comments', function (Blueprint $table) {
    $table->increments('id');
    $table->text('content');
    $table->unsignedInteger('user_id')->index();
    $table->unsignedInteger('post_id')->index();
    $table->unsignedInteger('like_count')->default(0);
    $table->unsignedInteger('reply_count')->default(0);
    $table->unsignedInteger('floor')->comment('楼层');
    $table->unsignedInteger('selected')->comment('是否精选')->default(0);
    $table->timestamps();
});

Schema::create('comment_replies', function (Blueprint $table) {
    $table->increments('id');
    $table->unsignedInteger('comment_id')->index();
    $table->unsignedInteger('user_id')->index();
    $table->text('content');
    $table->json('call_user')->nullable()->comment('@用户,{id: null, nickname: null}');
    $table->timestamps();
});
```

然后是用户表

```php
Schema::create('users', function (Blueprint $table) {
    $table->increments('id');
    $table->string('nickname');
    $table->string('avatar');
    $table->string('email');
    $table->string('phone_number');
    $table->string('password');

    $table->unsignedInteger('follow_count')->default(0)->comment("关注了多少个用户");
    $table->unsignedInteger('fans_count')->default(0)->comment("拥有多少个粉丝");
    $table->unsignedInteger('post_count')->default(0);
    $table->unsignedInteger('word_count')->default(0);
    $table->unsignedInteger('like_count')->default(0);

    $table->json('oauth')->nullable()->comment("第三方登录");

    $table->timestamps();
});
```

> 为什么从数据库上开始设计? 
>
> 从软件开发的角度来说,**数据是固有存在的,不会随着交互与设计的变化而变化**. 所以对于后端来说有了产品文档,就可以设计出接近完整的数据结构和80%左右的API了.



#### 建模

建立相关的[Model](https://learnku.com/docs/laravel/5.7/eloquent/2294) 及 [关联关系](https://learnku.com/docs/laravel/5.7/eloquent-relationships/2295)

> 这里需要多做一步,建立一个**[Model基类](https://learnku.com/docs/laravel-specification/5.5/data-model/503)**,其他的如 Post,Comment 继承自该 Model . 当然 我们不是要让 Model 成为一个 Super 类,只是通过该 Model 获得对其他 Model 的统一配置权. 

已 Comment 为例

```php
# Comment.php

<?php

namespace App\Models;


class Comment extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function replies()
    {
        return $this->hasMany(CommentReply::class);
    }

    public function post()
    {
        return $this->belongsTo(Post::class);
    }
}
```

其他的 Model 参考源码即可

#### 填充 Seeder

为了让前端更加顺畅的调试 api,seeder 是必不可少的一步. 接下来我们需要为上面建的几张表添加相应的 [factory](https://learnku.com/docs/laravel/5.7/database-testing/2304) 和 [seeder](https://learnku.com/docs/laravel/5.7/seeding/2292)

以 CommentFactory 为例

```php
# CommentFactory.php

<?php

use Faker\Generator as Faker;

$factory->define(\App\Models\Comment::class, function (Faker $faker) {
    static $i = 1;
    return [
        'post_id' => mt_rand(1, \App\Models\Post::count()),
        'user_id' => mt_rand(1, \App\Models\User::count()),
        'content' => $faker->sentence,
        'like_count' => mt_rand(0, 100),
        'reply_count' => mt_rand(0, 10),
        'floor' => $i++,
        'selected' => mt_rand(1, 10) > 2 ? 0 : 1
    ];
});
```

相应的 CommentSeeder

```php
# CommentSeeder.php

<?php

use Illuminate\Database\Seeder;

class CommentSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        factory(\App\Models\Comment::class, 1000)->create();
    }
}
```

其他的 factory 和 seeder 参考源码呀

> Seeder 的规范命名应该是  CommentsTableSeeder.php ,请不要学我!



## 发射

#### 确定 API

还是先来看看 [设计稿](https://www.jianshu.com/p/aff99d1d194b) , 来建立第一批 API

初步来看文章详情页分为三部分. 文章内容部分,评论回复部分,和推荐阅读部分.由于我们目前只建了几张基本表,所以先忽略推荐阅读部分.

API 设计的一个原则是同一个页面不要请求太多次 API ,否则会给服务器带来很大的压力.但也不能是一条非常聚合的api包含一个页面所有的数据. 这样则失去了 API 的灵活与独立性. 也不符合 RESTFul API 的设计思路

> **RESTFul API 是面向资源/数据的,是对资源的增删改查. 而不是面向界面/具体的业务逻辑**
>
> 所以上面说从设计稿切入实际是有些误导的,原则上是不需要设计稿的.这里的目的是为了推动文章向下进行,且能够更快的看到成果



按照 [tree-ql](https://github.com/weiwenhao/tree-ql) 的风格,我设计了这样两条 API

[test.com/api/posts/{post}?include=content,user.description,selected_comments ](http://community.eienao.com/api/posts/1?include=content,user.description,selected_comments)

[test.com/api/posts/{post}/comments?include=user,replies(limit:3).user](http://community.eienao.com/api/posts/1/comments?include=user,replies(limit:3).user)

> 上面的api是真实可以点击测试的,你可以随意修改include中的字段,来观察API的变化
>
> 执行请求的详细信息可以通过 [telescope](http://community.eienao.com/telescope) 查看

我们来解读一下上面两条 API

1. 取出帖子 `{post}`,并且包含该帖子的详情,用户(用户需要包含描述)和这篇帖子的所有精选评论
2. 取出帖子 `{post}`下的评论,并且每条评论需要包含相关用户和**回复/限制三条**(回复需要包含相关用户)

毫不知羞耻的说,上面的API是极其富于可读性的,并且有了 include 的存在,可控性也达到了非常高的地步

#### 路由

```php
# api.php

Route::get('posts/{post}', 'PostController@show');

Route::get('posts/{post}/comments', 'CommentController@index');
```



还要为`{post}`进行 [路由模型绑定](https://learnku.com/docs/laravel/5.7/routing/2253#2f0069)

```php
# RouteServiceProvider.php

public function boot()
{
    parent::boot();
    
    Route::bind('post', function ($value) {
        // columns的作用稍后会解释
        return Post::columns()->where('id', $value)->first();
    });
}
```



#### 控制器

由于没有做版本控制,所以没有添加类似`Api/V1`这样的目录. 以第二条 API 对应的控制器为例

```php
# app/Http/Controllers/Api/CommentController

<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Comment;
use App\Resources\CommentResource;
use Illuminate\Http\Request;
use Illuminate\Http\Response;

class CommentController extends Controller
{
    /**
     * @param null $parent
     * @return \Weiwenhao\TreeQL\Resource
     */
    public function index($parent = null)
    {
        // 1.
        $query = $parent ? $parent->comments() : Comment::query();
        
        // 2.
        $comments = $query->columns()->latest()->paginate();

        // 3.
        return CommentResource::make($comments);
    }
}
```

至此我们构成了 [test.com/api/posts/{post}/comments](http://community.eienao.com/api/posts/1/comments) 这条路由的访问控制器,但此时还不能include任何东西.在说明如何定义include之前,我们先对控制器中的三处标注进行讲解.

1. 进行了路由模型绑定的兼容处理,使得一个控制器可以兼容多条路由. 具体可以参考 [优雅的使用路由模型绑定](https://learnku.com/articles/17476)
2. 这是常见的 Builder 查询构造器,不严格讨论的话 `get() ∈ paginate()`, 因此使用适用范围更广的 paginate 作为结果输出.  columns 是一个查询作用域,由 tree-ql 提供,其赋予了精确查询数据库字段的能力.
3. 将查询的结果集交付给 Resource, 此 Resource 并非 laravel 原生的 [Resource](https://learnku.com/docs/laravel/5.7/eloquent-resources/2298),而是 **tree-ql 提供的 Resource** ,其会赋予我们 include 的能力,下面介绍一下该 Resource.

> 在阅读下面的内容之前你需要阅读一下 tree-ql 的文档



#### Resource

已 CommentResource 为例

```php
# CommentResource.php

<?php

namespace App\Resources;

use Weiwenhao\TreeQL\Resource;

class CommentResource extends Resource
{
    protected $default = [
        'id', 
        'content', 
        'user_id', 
        'like_count',
        'reply_count',
        'floor'
    ];

    protected $columns = [
        'id', 
        'content', 
        'user_id', 
        'post_id', 
        'like_count',
        'reply_count', 
        'floor'
    ];

    protected $relations = [
        'user',
        'replies' => [
            'resource' => CommentReplyResource::class,
        ]
    ];
}
```

其中 columns 代表着 comments 表的字段, relations 定义的内容为 代表 comment 模型中已经定义的关联关系.

API 请求中有些数据每次都需要加载,因此 **default 中定义的字段会被默认 include** ,而不需要在 url 中显式的定义.

由于 CommentResource 的 relations 部分还依赖了 user 和 replies ,按照 tree-ql 的规则我们需要分别定义 UserResource 和 RepliesResource.

```php
# UserResource.php

<?php

namespace App\Resources;

use Weiwenhao\TreeQL\Resource;

class UserResource extends Resource
{
    protected $default = ['id', 'nickname', 'avatar'];

    protected $columns = ['id', 'nickname', 'avatar', 'password'];
}

```



```php
# CommentReplyResource.php

<?php

namespace App\Resources;

use Weiwenhao\TreeQL\Resource;

class CommentReplyResource extends Resource
{
    protected $default = ['id', 'comment_id', 'user_id', 'content', 'call_user'];

    protected $columns = ['id', 'comment_id', 'user_id', 'content', 'call_user'];

    protected $relations = ['user'];

    /**
     * ...{post}/comments?include=...replies(limit:3)...
     *
     * ↓ ↓ ↓
     *
     * $comments->load(['replies' => function ($builder) {
     *      $this->loadConstraint($builder, ['limit' => 3])
     * });
     *
     * ↓ ↓ ↓
     * @param $builder
     * @param array $params
     */
    public function loadConstraint($builder, array $params)
    {
        isset($params['limit']) && $builder->limit($params['limit']);
    }
}
```

wo~, 我们已经完成了代码编写,客户端可以请求API了 …… 吗?

再来品味一下第二条api, **取出帖子 `{post}`下的评论,且每条评论携带3条回复,  ORM(MySQL) 可以做到这样的事情吗?** 

[点我看答案](https://learnku.com/articles/24787)

这里我选择了 PLAN C ,至此我们才算完成第二条api的编写,愉快的 [request](http://community.eienao.com/api/posts/1/comments?include=user,replies(limit:3).user) 吧

#### 但是

上面的API这么花里胡哨,会不会有性能问题?

来看看这条 API 的实际 SQL 表现,可以看到 SQL 符合预期,并没有任何的 n+1 问题,在速度方面可以说是有保障的. 实际上只要按照 tree-ql 的规范,无论多么花里胡哨的 include ,都不会有性能问题.

![](https://cdn.learnku.com/uploads/images/201903/12/10960/H9uZKWjav8.png!large)

> 调试工具 [laravel/telescope](https://github.com/laravel/telescope)



#### Workflow

走完了一套流程,稍微总结一下↓



![](https://cdn.learnku.com/uploads/images/201903/12/10960/QGYs8Eoz6m.png!large)

> Workflow 中去掉了确定 API 这一步,因为我们只要按照 RESTful 编写路由,按照 tree-ql 编写 Resource , API 自然而然的就出来啦~



## 补充

#### 文章详情API

[api.test.com/api/posts/{post}?include=content,user.description,selected_comments ](http://community.eienao.com/api/posts/1?include=content,user.description,selected_comments)

这里的 selected_comment 意为精选的评论,简书此处使用了单独的 api 来请求精选的评论.但是考虑到一篇帖子的精选评论通常不会太多.因此我采用 include 的方式 将精选评论与帖子一种返回.

**帖子和精选评论之间的的关系就是 data 和 meta 的关系**. 来看看相关的配置代码

```php
# CommentResource.php

<?php

namespace App\Resources;

use Weiwenhao\TreeQL\Resource;

class PostResource extends Resource
{
    protected $default = [
        'id',
        'title',
        'description',
        'cover',
        'comment_count',
        'like_count',
        'user_id'
    ];

    protected $columns = [
        'id',
        'title',
        'description',
        'cover',
        'read_count',
        'word_count',
        'give_count',
        'comment_count',
        'like_count',
        'user_id',
        'content',
        'selected_at',
        'published_at'
    ];


    protected $meta = [
        'selected_comments'
    ];


    public function selectedComments($params)
    {
        $post = $this->getModel();

        $comments = $post->selectedComments;

        // 这里的操作类似于 $comments->load(['user', 'replies.user'])
        // 但是load可不会帮你管理Column. 因此我们使用Resource来构造
        $commentResource = CommentResource::make($comments, 'user,replies.user');

        // getResponseData既获取CommentResource解析后并构造后的结构数组
        return $commentResource->getResponseData();
    }
}
```



#### 推荐阅读

[设计稿](https://www.jianshu.com/p/aff99d1d194b) 的最后一部分,分为两个小点. 分别是专题收录和推荐阅读.专题和帖子之间是多对多的关系.

推荐的做法比较丰富,简单且推荐的做法就是通过标签来推荐.但是这里我们有了专题这个概念后,其就充当了标签的概念.

下一篇会介绍专题与推荐阅读的一些需要注意的细节.
