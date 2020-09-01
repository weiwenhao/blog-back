---
title: '编写具有描述性的 RESTful API (三): 用户行为'
date: 2019-03-20 10:52:09
tags:
---

用户行为泛指用户在网站上的交互,如点赞/关注/收藏/评论/发帖等等. 这些行为具有**共同的特征**,因此可以用**一致的编码**来描述用户的行为

## 用户行为分析

我们先来分析一下简书中的用户行为, user_follow_user (用户关注用户) / user_like_comment / user_like_post / user_give_post (用户赞赏帖子) / user_follow_collection (用户关注专题)/ user_comment_post 等等

按照 RESTful 规范将上面的行为抽象为资源, 可以得到 user_follower , comment_liker , post_liker, post_giver , collection_follower 等资源.

> 这里我忽略了一个可能的资源 post_commenters , 因此它已经存在了,就是 comments 表, 因此不做考虑

接下来以 post_liker 为例来看一下实际的编码

#### 建表

```php
Schema::create('post_likers', function (Blueprint $table) {
  $table->increments('id');
  $table->unsignedInteger('post_id');
  $table->unsignedInteger('user_id');
  $table->timestamps();

  $table->index('post_id');
  $table->index('user_id');

  $table->unique(['post_id', 'user_id']);
});
```

#### 建模

这里 post_liker 已经被提取成了资源的形式,虽然其依旧充当着中间表的作用,但形式上需要按照资源的形式来处理.

即表名需要复数形式,且需要相应的模型与控制器. 概言之

`> php artisan make:model Models/PostLiker -a`

```php
<?php

namespace App\Models;

class PostLiker extends Model
{
    public function post()
    {
        return $this->belongsTo(Post::class);
    }

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

#### 路由

 post 请求表示创建 liker 资源 即用户喜欢帖子时需要调用该 API ,反之 delete 则表示删除 liker 资源.

```php
# api.php

Route::post('posts/{post}/liker', 'PostLikerController@store');
Route::delete('posts/{post}/liker', 'PostLikerController@destroy');
```



#### 控制器

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Post;
use App\Models\PostLiker;
use Illuminate\Http\Response;

class PostLikerController extends Controller
{
    /**
     * @param Post $post
     * @return \Illuminate\Contracts\Routing\ResponseFactory|Response
     */
    public function store(Post $post)
    {
        $postLiker = new PostLiker();
        $postLiker->user_id = \Auth::id();
        $postLiker->post()->associate($post);
        $postLiker->save();

        return $this->created();
    }


    /**
     * @param Post $post
     * @return \Illuminate\Contracts\Routing\ResponseFactory|Response
     */
    public function destroy(Post $post)
    {
        $postLiker = PostLiker::where('user_id', \Auth::id())
            ->where('post_id', $post->id)
            ->firstOrFail();

        // Observer 中需要用到该关联,防止重复查询
        $postLiker->setRelation('post', $post);

        $postLiker->delete();

        return $this->noContent();
    }
}

```

至此我们就成功把喜欢帖子的行为转换成了 RESTful API.

>  destroy 方法为了触发 observer 显得有些繁琐,还有待优化. 

#### Observer

在 Observer 中,我们处理了与主逻辑不想关的附加逻辑, 即增减 post.like_count

```php
<?php

namespace App\Observers;

use App\Models\PostLiker;

class PostLikerObserver
{
    /**
     * @param PostLiker $postLiker
     */
    public function created(PostLiker $postLiker)
    {
        $postLiker->post()->increment('like_count');
    }

    public function deleted(PostLiker $postLiker)
    {
        $postLiker->post()->decrement('like_count');
    }
}

```

还没完,还有一步重要的实现,便是**判断当前用户是否已经喜欢了该帖子**,或者说在一个帖子列表中当前用户喜欢了哪些帖子

#### Is like

为了进一步增加描述性和前端绑定数据的便利性. 我们需要借助 tree-ql 来实现该功能

```php
# PostResource.php

<?php

namespace App\Resources;

use Illuminate\Support\Facades\Redis;
use Weiwenhao\TreeQL\Resource;

class PostResource extends Resource
{
  	// 单次请求简单缓存
    private $likedPostIds;

   // ...


    protected $custom = [
        'liked'
    ];


    /**
     * @param $post
     * @return mixed
     */
    public function liked($post)
    {
        if (!$this->likedPostIds) {
            $this->likedPostIds = $user->likedPosts()->pluck('posts.id')->toArray();
        }

        return in_array($post->id, $this->likedPostIds);
    }
}
```

定义好之后,就可以愉快的 include 啦

[test.com/api/posts?include=liked](http://community.eienao.com/api/posts?include=liked)

[test.com/api/posts/1?include=liked](http://community.eienao.com/api/posts/1?include=liked)

文末会附上 Postman 文档.**其余用户行为,同理实现即可.**



## 统一用户行为

用户行为既然具有一致性,那么其实可以进一步抽象来避免大量的用户行为控制器和路由.

完全可以使用一个 UserActionController , 前端配套 userAction.js  来统一实现用户行为,

更进一步的想法则是,**前端在 localStorate 中维护一份用户行为的资源数据,可以直接通过 localStorate 来判断当前用户是否喜欢过某个用户/是否喜欢过某篇文章等等,从而减轻后端的计算压力.**

> localStorage 中的资源示例
>
> post_likers: [12, 234, 827, 125]
>
> comment_likers: [222, 352, 122]

下面的示例是我曾经在项目中按照该想法实现的 userAction.js

```php
import Vue from 'vue'
import userAction from '~/api/userAction'

/**
 *  用户行为统一接口
 *
 *  由于该类使用了浏览器端的 localStorage 所以只能在客户端调用
 *  actionName 和对应的 primaryKey 分别为
 *  follow_user | 被关注的用户的 id - 关注用户行为
 *  join_community | 加入的社区的 id - 加入社区行为
 *  collect_product | 收藏的产品的 id - 收藏产品行为
 *
 * +----------------------------------------------------------------------------
 *
 *  推荐在组件的 mounted 中调用 action 中的方法.已保证调用栈在浏览器端
 *  is() 方法属于异步方法,返回一个 Promise 对象,因此需要使用 await承接其结果
 *
 * +----------------------------------------------------------------------------
 *  mounted 中的调用示例. 判断当前用户是否关注了用户 id 为 1 的用户.
 *  async mounted () {
 *    this.isAction = await this.$action.is('follow_user', 1, this)
 *  }
 */
Vue.prototype.$action = {
  /**
   * 是否对某个对象产生过某种行为
   * @param actionName
   * @param primaryKey
   * @param vue
   * @returns {Promise<boolean>}
   */
  async is (actionName, primaryKey, vue) {
    if (!vue.$store.state.auth.user) {
      throw new Error('用户未登录')
    }
    
    if (!primaryKey) {
      throw new Error('primaryKey必须传递')
    }
    
    primaryKey = Number(primaryKey)
    
    // 如果本地没有在 localStorage 中发现相应的 actionName 则去请求后端进行同步
    if (!window.localStorage.getItem(actionName)) {
      let {data} = await vue.$axios.get(userAction.fetch(actionName))
        
      Object.keys(data).forEach(function (item) {
        localStorage.setItem(item, JSON.stringify(data[item]))
      })
    }

    let item = localStorage.getItem(actionName)
    item = JSON.parse(item)
    if (item.indexOf(primaryKey) !== -1) {
      return true
    }

    return false
  },

  /**
   * 创建一条用户行为
   * @param actionName
   * @param primaryKey
   * @param vue
   * @returns {boolean}
   */
  create (actionName, primaryKey, vue) {
    if (!vue.$store.state.auth.user) {
      throw new Error('用户未登录')
    }
    
    if (!primaryKey) {
      throw new Error('primaryKey必须传递')
    }

    primaryKey = Number(primaryKey)
    
    // 发送 http 请求
    vue.$axios.post(userAction.store(actionName, primaryKey))

    // localStorage
    let item = window.localStorage.getItem(actionName)
    if (!item) {
      return false
    }
    item = JSON.parse(item)
    if (item.indexOf(primaryKey) === -1) {
      item.push(primaryKey)
      localStorage.setItem(actionName, JSON.stringify(item))
    }
  },

  /**
   * 删除一条用户行为
   * @param actionName 用户行为名称
   * @param primaryKey 该行为对应的primaryKey
   * @param vue
   * @returns {boolean}
   */
  delete (actionName, primaryKey, vue) {
    if (!vue.$store.state.auth.user) {
      throw new Error('用户未登录')
    }
    
    if (!primaryKey) {
      throw new Error('primaryKey必须传递')
    }

    primaryKey = Number(primaryKey)

    // 发送 http 请求
    vue.$axios.delete(userAction.destroy(actionName, primaryKey))

    let item = window.localStorage.getItem(actionName)
    if (!item) {
      return false
    }
    item = JSON.parse(item)
    let index = item.indexOf(primaryKey)
    if (index !== -1) {
      item.splice(index, 1)
      localStorage.setItem(actionName, JSON.stringify(item))
    }
  }
}
```

理论上按照该想法来实现,可以极大前后端的编码压力.

相应的后端代码就类似上面的 PostLikerController, 只是前端请求中增加了action_name 做数据导向.



## 补充

#### Redis缓存

用户行为的存储结构非常符合 Redis 的 SET 数据结构, 因此来尝试使用 set 对用户行为进行缓存,依旧使用 post_likers 作为例子. 

Resource

```php
<?php

namespace App\Resources;

use App\Models\Post;
use Illuminate\Support\Facades\Redis;
use Weiwenhao\TreeQL\Resource;

class PostResource extends Resource
{
   // ...

    protected $custom = [
        'liked'
    ];


    /**
     * @param Post $post
     * @return boolean
     */
    public function liked($post)
    {
        $user = \Auth::user();
        $key = "user_likers:{$user->id}";

        if (!Redis::exists($key)) {
            $likedPostIds = $user->likedPosts()->pluck('posts.id')->toArray();

            Redis::sadd($key, $likedPostIds);
        }

        return (boolean)Redis::sismember($key, $post->id);
    }
}
```

相关的更新和删除缓存在 Observe中进行,控制器编码无需变动

```php
# PostLikerObserver.php

<?php

namespace App\Observers;

use App\Models\PostLiker;
use Illuminate\Support\Facades\Redis;

class PostLikerObserver
{
    /**
     * @param PostLiker $postLiker
     */
    public function created(PostLiker $postLiker)
    {
        $postLiker->post()->increment('like_count');

        // cache add
        $user = \Auth::user();
        $key = "user_likers:{$user->id}";
      
        if (Redis::exists($key)) {
            Redis::sadd($key, [$postLiker->post_id]);
        }
    }

    public function deleted(PostLiker $postLiker)
    {
        $postLiker->post()->decrement('like_count');

        // cache remove
        $userId = \Auth::id();
        $key = "user_likers:{$userId}";

        if (Redis::exists($key)) {
            Redis::srem($key, $postLiker->post_id);
        }
    }
}
```



#### 更进一步

上面的做法是我过去在项目中使用的方法,但是在编写该篇时,我的脑海中冒出了另外一个想法

[test.com/api/posts?include=liker](http://community.eienao.com/api/posts?include=liker)

稍微去源码看看 PostResource 便明白了. 使用该方法能够优化上文提到的 destroy 方法过于繁琐的问题.

#### 相关

- Api document [https://documenter.getpostman.com/view/1500624/S17nTVKL](https://documenter.getpostman.com/view/1500624/S17nTVKL)
- 线上调试工具 [telescope](http://community.eienao.com/telescope)
- 本节源码 [weiwenhao/community-api](https://github.com/weiwenhao/community-api/tree/0.3.0-alpha)
- Api Tool [weiwenhao/tree-ql](https://github.com/weiwenhao/tree-ql)
- 上一节 [编写具有描述性的 RESTful API (二): 推荐与 Observer](https://learnku.com/articles/25444)

