---
title: 编写更具有描述性的 RESTful API
date: 2018-04-01 15:36:09
tags:
---

我时常觉得后端应该关心的是数据，而不是业务。
因此我希望能够在数据的基础上编写一套接口 能够满足h5端、pc端、ios/android端、包括小程序端等等80%的需求

<!-- more -->

laravel + dingo/api 对于api开发来说已经足够友好了，因此选择在它的基础上构建。

预备的知识

- laravel
- RESTful
- dingo/api

## 查询Filter

### 排序

排序字段的选择在网上有很多种

 1. `?sort_field=created_at&sort_order=asc ` 
 2. `?order=+created`
 3. `?sort_by=created_at&order=asc`
 ...
 
 这里选择在第三种方式,在可读性和数据处理上更加方便
 
**url：** `http://api.test/api/posts?sort_by=created_at&order=desc`

**laravel：** `$query->orderBy(request()->get('sort_by', 'id'), request()->get('order', 'desc'))`

### 分页

对于分页来说 `offset/limit`和`page/per_page` 又是两个纠结的选择
常见的分页需求有两种，一种时普通的ajax分页，另外一种是下拉加载更多分页
ajax分页相比于加载更多 通常需要一个total

因此选择能同时适应两种分页需求的 `page/per_page`，laravel和dingo/api对该方式的支持也足够好


**url：** `http://maxwei.me/api/posts?per_page=3&page=2`

**laravel：** `$query->paginate(request()->get('per_page', 15))->appends(request()->except('page'))`

**返回结果中的links示例：**
```
"links": {
        "next": "http://api.test/api/posts?order=asc&page=3",
        "previous": "http://mp.test/api/diaries?order=asc&page=1"
},
```

### 字段筛选

**api：** `api.test/users?fields=id,nickname,avatar` 对于user表中的phone，password字段推荐使用Model的hidden属性隐藏。
**laravel：** `!is_null(request()->get('fields')) && $query->addSelect(explode(',', request()->get('fields')));`

transform()的常用写法和fields有一定的冲突，还没有找到比较优雅的解决方案。

### where筛选
当我们只想要状态为1的文章时 我希望可以这么做
**url：** `http://api.test/api/posts?status=1`

当我想要标签id为1, 2的文章时则这样
**url：** `http://api.test/api/posts?tag_id=1,2`

当我... 够了，简单点是我所追求的，我不希望去创建一些规则满足模糊查询、notIn、orWhere、嵌套where等等。这不具有通用性，如果需要可以创建一些特定的路由去满足这些条件即可。


**laravel:**

```php
$where = ['status', 'tag_id'] //这是我希望能够被筛选的字段

foreach ($where as $item) {
    if (is_null($value = request()->get($item))) {
        continue;
    }

    if (str_contains($value, ',')) {
        $query->whereIn($item, explode(',', $value));
    } else {
        $query->where($item, '=', $value);
    }
}
```

## 资源嵌套

有如下两种需求场景

- 获取某个用户/或标签下的所有文章
- 获取首页的精选文章

我希望这两种情况都能通过一个index()方法得到解决，因此我这样做

```php
    #api.php
    
    // 主资源路由 
    $api->resource('posts', 'PostController')
    
   // 多对多关系嵌套路由
    $api->get('tags/{tag}/posts', function ($id) {
        $tags = \App\Models\Tag::findOrFail($id);
        return app()->call('App\Http\Controllers\Api\PostController@index', ['query' => $tags->posts()]);
    });
    // 一对多关系嵌套路由
    $api->get('users/{user}/posts', function ($id) {
        $user = \App\Models\User::findOrFail($id);
        return app()->call('App\Http\Controllers\Api\PostController@index', ['query' => $user->posts()]);
    })
    
    // 上面的代码非常的有规律，可以进行一次封装，而不是这样不行的重复解析。laravel5.6支持的路由模型注入是个不错的注意，但是dingo/api目前还不支持
    // 别忘了在你模型中定义相应的关联关系
    
    
    # PostController.php
    
    public function index($query = null)
    {
        // parseFilter是我封装的一个用来解析通用参数的方法
        $paginator = $this->parseFilter($query ?? Post::query());
        return $this->response->paginator($paginator, new PostTransformer());
    }
    
```


## 资源关联
dingo/api + Fractal 对资源关联处理非常优雅，并且很好的解决了n+1 问题。

假设两个需求

- 当我取出多个文章资源时我希望能够关联它们的作者。

**url：** `http://api.test/posts?include=user:field(id|name|avatar)`

- 取出一个社区资源并附带几名活跃的用户资源，以及这些活跃用户最近发表过的3篇文章时
**url：** `http://api.test/hubs/1?include=hot_users:limit(3).posts:fields(id|title):limit(3)`

这大概就是我非常喜欢fractal而迟迟不肯使用laravel5.5的resources的原因， 因为它制定出了一套include的规则和相应的代码处理，使得代码的偶合性非常低。

> include参数的详细使用方式 请参考dingo/api文档 和 fractal文档 [https://fractal.thephpleague.com/](https://fractal.thephpleague.com/)

对于上面的需求我们可以这么做

```php
# PostTransformer.php

public function includeUser(Post $post, ParamBag $params)
{
    // fractal会帮我们解析include中的参数，并注入到 $params中。因此我们直接使用
    $user = $post->user()->select($params[fields] ?? '*')->firstOrFail();
    return $this->item($user, new UserTransformer());
}


# HubTransformer.php

public function includeHotUsers(Hub $hub, ParamBag $params)
{
    $users = $hub->hot_users()
        ->limit($params['limit'][0] ?? 5)
        ->get();

    return $this->collection($users, new UserTransformer());
}

# UserTransformer.php

public function includePosts(User $user, ParamBag $params)
{
    $post = $user->posts()
        ->select($params[fields] ?? ['id', 'title', 'description', 'like_count'])
        ->limit($params['limit'][0] ?? 5)
        ->get();
        
        return $this->collection($posts, new PostTransformer());
}

```


补充一下， 对于使用了 `$this->response()->collection()`和`$this->response->paginator()` 方法的资源。 dingo/api 会去解析url中的include参数，然后去调用模型的相应的关联方法来进行预加载，从而解决查询的n+1问题

上面的第二个需求，要求Hub模型中必须定义 hot_users和posts 这两个关联方法，否则就会抛出异常

> 这里模型定义的关联方法的名称必须与url一致 既 hot_users()。非常难受呀，因为url推荐小写，方法名推荐小驼峰！！



### 关联资源的参数过滤规则

`:参数名称(值1|值2|值N)`
':' 冒号标志着一个参数的开始
紧跟着是参数名称
然后接上参数值 其中参数的值需要被括号括起
多个参数值时使用 '|' 分隔

> 关联资源我并不推荐提供分页参数，因为其会造成数据的重复读取，如果需要取出的关联资源数据量很多。推荐通过单独的api请求获取该资源，而不是通过include方式加载进来。


## 资源中的动作

我们对资源存在一些动作行为，如对帖子的点赞收藏等，这里我选择模仿github的做法，将动作转换为资源。

### 创建与删除动作资源

- 点赞文章

**url:** `http://mp.test/posts/1/likes`
**method:** `POST`

- 取消点赞文章

**url:** `http://mp.test/posts/1/likes`
**method:** `DELETE`

```php
// PostLikeController.php
public function store(Request $request, $id)
{
    DB::table('user_like_post')->insert([
        'user_id' => \Auth::id(),
        'post_id' => $id
    ]);

    return $this->response->created();
}

public function destroy($id)
{
    DB::table('user_like_post')->where('user_id', \Auth::id())->where('post', $id)->delete();

    return $this->response->noContent();
}

```

### 验证动作资源
这是一个我研究/纠结了很久的问题，尝试过很多种写法，这里决定模仿知乎的api写法

验证用户是否点赞了某一篇帖子

**url：** `http://api.test/posts/1?include=is_like`

对于上面的url，dingo/api 会自动调用PostTransformer的includeIsLike方法。我们只需要在该方法中进行验证即可

```php
# PostTransformer.php

public function includeIsLike(Post $post)
{
    // 这行代码可以根据Auth::id Cache一下
    $likePostIds = DB::table('user_like_post')->where('user_id', Auth::id())->pluck('post_id')->toArray();

    return $this->primitive(in_array($diary->id, $likePostIds));
}

```
吐槽一下 includeIsLike如何返回标量资源，文档上没有任何描述。
看了源码才发现primitive这个关键词。😣


对于单个资源可以很容易的完成上面的需求，但对于资源集合我遇到了很大的问题

**url：** `http://api.test/posts?include=is_like`

集合我统一使用了`$this->response->paginator()`， 前面提到 paginator和collection方法，会去检测include参数并调用模型的相应的关联方法来进行预加载。 所以会去posts模型去找is_like方法，可是我真的定义不出一个is_like关联关系呢。
而且这个行为是没法优雅的禁止掉的，想要禁止？ok啊，那就全关了，别想我再给你解决n+1问题了

这明明是一个很容易解决的问题，在dingo/api的issue中也提到了多次。但是都没有得到解决。

> 于是我fork下了dingo/api的代码准备解决一下这个问题时，我终于明白是为什么了~
> dingo/api和Fractal是不同作者的项目。dingo/api是为laravel量身打造的。其依赖的transform使用的是Fractal。 而Fractal并不专属于laravel. 
> 在dingo/api中做很容易，但是在Fractal中添加一个为laravel服务的扩展就有些不切实际了

既然如此就在我们的项目中稍微解决一下这个问题

```php
# 定义一个根Transformers.php 所有的Transformer都继承自该Transformer
<?php

namespace App\Http\Transformers;

use League\Fractal\TransformerAbstract;

class Transformer extends TransformerAbstract
{
    protected $disableEagerLoadedIncludes = [];

    public function getDisableEagerLoadedIncludes()
    {
        return $this->disableEagerLoadedIncludes;
    }
}


# 定义一个 Fractal.php 并继承于原有Fractal

<?php

namespace App\Services;

class Fractal extends \Dingo\Api\Transformer\Adapter\Fractal
{
    protected function mergeEagerLoads($transformer, $requestedIncludes)
    {
        $includes = array_merge($requestedIncludes, $transformer->getDefaultIncludes());
        $includes = array_diff($includes, $transformer->getDisableEagerLoadedIncludes());
        $eagerLoads = [];

        foreach ($includes as $key => $value) {
            $eagerLoads[] = is_string($key) ? $key : $value;
        }

        return $eagerLoads;
    }


}


# 修改 dingo/api的配置文件api.php 将原有的Fractal更改为我们自定义的

'transformer' => env('API_TRANSFORMER', \App\Services\Fractal::class),

```
大功告成~ 接下来我们只需要在PostTransformer中定义一个 disableEagerLoadedIncludes属性来添加不需要急切加载的属性了。

```php
protected $disableEagerLoadedIncludes = ['is_like'];

```

终于可以  `?include=is_like,like_count,is_author,balala...` 面向include的编程了

## 补充

上面说的都是已查询为主，但增删改也有一些小技巧 如创建资源除了使用 dingo/api封装的`$this->response->created() `外， 使用`$this->response->item($post, new PostTransformer())->setStatusCode(201);`也是一种不错的选择。

使用 request进行表单验证、使用 Policiy进行权限验证、使用Observer进行副作用的处理等等，从而保证增删改的代码更具有可读性和解耦性。


另外还有很多待解决的问题

- 如嵌套资源中，直接在路由文件中处理逻辑并不优雅
- fields 筛选字段对 Transform的传统写法并不友好 有待改进
- fields方法在laravel中显得和include有些冲突，是否可以直接在include中编写需要获取的fields呢
    - 这里我觉得可能需要抛弃原有transform()的写法，数据处理应该通过orm提供的一些修改器来进行。更多的数据处理则交给客户端，服务端提供一些更加raw的数据。
- 当一个界面过于复杂时，需要请求多次api，超过3次以上我就有些难以接受了，get请求url过长问题，接下来会尝试进行路由的映射 组装操作等操作解决这个问题
    
- 资源暴露是否会带来安全问题？这一点我觉不会。如何认证和限流参考dingo/api文档即可
- 对于一些不符合RESTful资源的需求如何处理，如搜索需求。可以尝试创建些额外的路由来处理这些额外的需求。这些需求可以占到一个项目的20%左右
- 资源控制器的index方法使用的page每次都会多查询一条总记录数的sql。
- 资源控制器的index方法如果不存在筛选条件时如何做资源限制，总不能一次取出所有的资源
    - 突发奇想采用了一个sql黑名单的机制
         
        ```php
        private function isBlacklist($query)
        {
                $limit = 100;

                if (request('per_page') && request('per_page') < $limit) {
                        return false;
                }

                $key = 'sql:'. $query->toSql();

                if (Cache::has($key)) {
                        return true;
                }
                if ($query->count() > $limit) {
                        Cache::forever($key, date('Y-m-d H:i:s'));
                        return true;
                }

                return false;
        }
        ```
         
...

## 结语

再次说明一下我在做什么，我希望能够在数据的基础上编写一套接口能够满足h5端、pc端、ios/android端、包括小程序端等等80%的需求。
我不希望我的接口要跟着每一次的业务变动而去修改，我希望自己关心的是数据，而不是业务。
我希望只要知道产品的原型，就能完成后端80%的开发， 而不是等设计定稿/等前端开发等等


现在前端的开发是模块化的，在我看来就是面向import的开发。
传统的RESTful接口没法适应于前端的模块化开发。存在着大量的字段冗余和http请求，这是我在学习graphQL的时看到的一句话。

但既然前端的开发是模块化的、面向import的，后端的api接口为什么不能是面向include的呢 😆

> 持续关注中 - 希望你能分享在RESTful API、dingo/api、laravel api等开发时的经验、想法和技巧~
> 我将会总结出一份代码demo并分享出来
