---
title: 优雅的使用路由模型绑定
date: 2018-09-21 15:44:20
tags: laravel api
---

## Basic

> laravel5.1/5.2发布的路由模型绑定是一个非常强大的功能,dingo/api中想要使用路由模型绑定需要引入bindings组件到dingo的路由组中

<!-- more -->

#### 路由中的参数命名

如有需要,使用资源的单数形式为路由命名,而不是 {id}/{slug}/{code}等等

正确的示例

```php
# api.php

Route::resource('posts', 'PostController'); // laravel会将参数命名为 post
Route::get('users/{user}/posts', 'PostController@index');
Route::get('posts/{post}/comments', 'CommentController@index');
```



#### 显式绑定

以[简书](https://www.jianshu.com/)的文章为例,使用RESTFul 风格可以得到以下几条路由

```php
# api.php

Route::get('posts', 'PostController@index'); // 首页/基础帖子展示
Route::get('users/{user}/posts', 'PostController@index'); // 某个用户的帖子
Route::get('collections/{collections}/posts', 'PostController@index') // 某个专题下的帖子
```



可以看到,这里我使用了同一个方法来处理这三条路由.

接下来定义路由模型绑定,这里使用显式绑定,以获得较大的灵活性

```php
# RouteServiceProvider.php

Route::model('user', User::class);
Route::model('collection', Collection::class)
```



然后看看控制器中的定义

```php
# PostController.php

public function index($parent = null)
{
    $query = $parent ? $parent->posts() : Post::query();

    // e.g
    $posts = $query->latest()->paginate();

    // ...
}
```



当我们使用第一条路由访问我们的帖子时, $parent得到的是一个null, `$query = Post::query`.

访问后两条路由时,由于路由模型绑定,$parent 被赋值为具体的model. 此时可以通过model中定义的关联关系来获取query. 通过显示绑定和关联关系的定义,使得$parent->posts()足够抽象,不依赖于具体的model.具有强大的通用性.



#### p.s 

```php
# User.php
 
public function posts()
{
    return $this->hasMany(Post::class);
}

```

```php
# Collection.php

public function posts()
{
    return $this->belongsToMany(Post::class, 'collection_post');
}
```



## Advance

对于, 如 我的文章列表,我的订单等我们可能会这样定义我们的 RESTFul 路由

```php
// 这里单数形式的user就代表着me的意思, 参考于github api
Route::get('user/posts', 'PostController@index');
Route::get('user/orders', 'OrderController@index');
```



甚至,依旧已简书为例,简书有两个板块[30日热门](https://www.jianshu.com/trending/weekly?utm_medium=index-banner-s&utm_source=desktop)和[7日热门](https://www.jianshu.com/trending/monthly?utm_medium=index-banner-s&utm_source=desktop), 我们可能会有这样两条路由

```php
Route::get('hot-30/posts', 'PostController@index');
Route::get('hot-7/posts', 'PostController@index');

// 又或者根据推荐算法通过用户画像推荐不同的文章
Route::get('recommend/posts', 'PostController@index');
```

> 不要激动,接下来不是算法环节?

?现在的问题是,如何依旧使用同一个方法实现?的几条路由呢?

**我们换一种思路.前面我们都是在控制器层面做抽象,然后把具体逻辑交给路由层.可面对上面的需求依旧有些力不从心,那我们不妨寻找一下上面需求的共同点,再提取一层抽象**

> 当然更简单的思路是 拆开几个方法写就ok啦,搞这么麻烦是吧.见仁见智.瞎折腾就是了

我们可以这么做

```php
# api.php

// 使用 {virtual} 来匹配上面的hot-30,hot-7,recommend 等等
Route::get('{virtual}/posts', 'PostController@index'); 
```



```php
# RouteServiceProvider.php

Route::bind('virtual', function ($value) {
    $virtual = "App\\Virtual\\" . studly_case($value);
    return new $virtual($value);
});
```

上面的做法是, 路由模型绑定是基于model,或者说 entity 的*(在symfony中model被称作entity)*.但是hot-30/hot-7/recommend 并不基于model.*(当然也可以基于model,不过这不是我们本次讨论的重点)*, 

那我们不妨使用一个virtual 来承载它们, virtual是一个和entity相近又相反的意思.在这里再适合不过了.来看看具体实现

```php
# Hot30.php

namespace App\Virtual;

use App\Models\Post;

class Hot30 extends Virtual
{
    public function posts()
    {
        $ids = ...; // service
        return Post::whereIn('id', $ids);
    }
}
```

Hot7,Recommend同理. 这样我们又承接起了上面控制器的代码.这里的posts()的作用就相当于比如User.php中的posts()的作用,但是却更加的灵活.

> 鲁迅说过: 不要害怕在你的app下添加目录


我只是分享了一个简单的想法,更多的用法等着你来探索.

successful!


