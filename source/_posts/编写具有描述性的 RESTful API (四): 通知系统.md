---
title: '编写具有描述性的 RESTful API (四): 通知系统'
date: 2019-03-27 12:40:09
tags:
---

上一节讲到了用户行为，由用户行为自然的便引出了通知系统。当用户喜欢了一篇帖子，那么该帖子的作者应该收到一条提醒。

laravel 提供了一套 [Notification](https://learnku.com/docs/laravel/5.8/notifications/3921) 组件，用于处理通知，其支持通过多种频道发送通知，包括邮件、短信 、通知还能存储到数据库（站内信）以便后续在 Web 页面中显示，**本节将重点放在站内信。**

laravel 为我们摆平了通知的推送问题，但是还有一个问题，即通知（数据库）的存储问题需要我们处理。通知的存储通常有两种做法。

- 将通知的主体内容与部分附加内容一同冗余存储到数据库中，即 laravel 的默认形式。
- 将通知的主体内容的主键与其附加内容的主键 既 id 存储到数据库中。

前者的好处是较小的查询压力，且数据具有持久性，不会因为被删帖等问题而影响到通知内容。缺点则是占用存储空间，且缺乏灵活性，后者则反之。

源码中选择了前者，既默认形式。

## 数据分析

通过抽象可以得到，一条通知由三部分组成 **行为的触发者 trigger 、行为主体(可能携带 内容) target 、需要通知的用户 notifiable**

此处最让人疑惑的应该是 行为的主体，根据实际需求稍微图解一下。

![](https://cdn.learnku.com/uploads/images/201903/27/10960/gyqloOm9E0.png!large)



已 「Comment Post」为例，在简书中其实际的表现行为如下

![](https://cdn.learnku.com/uploads/images/201903/27/10960/Vc0YhVnK7v.png!large)

根据上面的分析 notifications 表中的 data 需要冗余如下数据，**可以根据运营的实际需求调整**。

```php
public function toArray($notifiable)
 {
   $data = [
     'trigger' => [
       'id' => 1, // default type users
       'type' => 'users',
       'nickname' => 'nickname',
       'avatar' => 'xxx',
     ],
     'target' => [
       'id' => 12,
       'type' => 'posts',
       'text' => 'xxx',
     ],
     'content' => [
       'id' => 1,
       'type' => 'comment',
       'text' => 'xxx',
       'call_user' => [
         'id' => 'xxx',
         'nickname' => 'xxx'
       ]
     ]
   ];
  
   return $data;
 }
```



## 创建通知

已上一次的 「Like Post」 行为为例，当用户点赞文章后，需要给文章的作者发送一条通知。

```php
# PostLikerObserver


/**
 * @param PostLiker $postLiker
 */
public function created(PostLiker $postLiker)
{
  // ...

  // notify App\Notifications\LikePost
  User::findOrFail($postLiker->post->user_id)
    ->notify(new LikePost($user, $postLiker->post));
}
```

```php
# LikerPost.php

namespace App\Notifications;


class LikePost extends Notification
{
    use Queueable;
   
    private $post;
    private $trigger;
    private $target;

    public function __construct(User $trigger, Post $target)
    {
        $this->trigger = $trigger;
        $this->target = $target;
    }

    // ...
  
    public function toArray($notifiable)
    {
        $data = [
            'trigger' => [
                'id' => $this->trigger->id,
                'type' => $this->trigger->getTable(),
                'nickname' => $this->trigger->nickname,
                'avatar' => $this->trigger->avatar,
            ],
            'target' => [
                'id' => $this->target->id,
                'type' => $this->target->getTable(),
                'text' => $this->target->title,
            ]
        ];

        return $data;
    }
}
```

这样就成功建立了一条 Notification ,  类似「Comment Post」等用户行为依旧可以按照这种思路完成。无非「Comment Post」需要在其 data 中添加 content 而已，这里就不做展示了。

这里需要提一下代码优化，通过上面的「数据分析」，我们已经把通知抽象为 trigger / target / content ，因此并不需要再每种用户行为都编写一堆  重复的构造方法，toArray 等方法。

完全可以编写一个 Notification 基类来编写上面的大部分代码，从而减少重复的代码。

> 相关优化已经完成，欢迎参考源码。



## 通知的另一种存储方式

这里稍微提一下另外一种形式的通知存储，即上文中提到的非冗余形式。不过该方式需要修改 notifications 表的默认表结构。

```php
Schema::create('notifications', function (Blueprint $table) {
  $table->uuid('id')->primary();
  $table->string('type');
  $table->morphs('notifiable');
  
  // $table->json('data')->nullable()->comment('target/content/trigger');
  $table->morphs('triggerable')->nullable();
  $table->morphs('targetable')->nullable()
  $table->morphs('contentable')->nullable();
  
  $table->timestamp('read_at')->nullable();
  $table->timestamps();
});
```

通过表结构相信你已经一目了然。万变不离其宗，**我们始终都是在围绕着 trigger / target / content 转圈圈。**

> 当然如果你了解 「 MySQL 生成列 」的话，完全可以写出下叙语句，将通知从冗余形式平滑过渡到到非冗余形式。
>
> `$table->string('targetable_id')->virtualAs('data->>"$.target.id"')->index();`

有了这样的表结构，关联关系走起来，需要的数据如 文章的 点赞量 / 阅读量 等等都能够得到，这里就不详细描述代码编写了。后续简书的**用户动态模块**会再次运用这种非冗余结构的编码，到时再深入讲解相关的细节。



## 通知压缩

我们的通知采用了冗余形式存储，所以数据存储空间优化是必须考虑的一个点。尤其是在点赞这类通知中，对数据的浪费是非常巨大的。

因此可以通过类似下面这样的方式压缩**未读通知**。

![](https://cdn.learnku.com/uploads/images/201903/27/10960/pOemJX9z1m.png!large)

> 当然，如果你选择非冗余形式存储通知数据，那么将难以进行数据压缩。

Notification 组件提供了通知后 [事件](https://learnku.com/docs/laravel/5.8/notifications/3921#notification-events) ,所以相应的逻辑将会在该事件的监听者中完成。逻辑比较简单，直接看编码吧

```php
# App\Listeners\CompressNotification

class CompressNotification
{
    public function handle(NotificationSent $event)
    {
        $channel = $event->channel;

        if ($channel !== DatabaseChannel::class) {
            return;
        }

        $currentNotification = $event->response;
        if (!in_array($currentNotification->type, ['like_post', 'like_comment'])) {
            return;
        }

        $notifiable = $event->notifiable;
      
      	// 查找相同 target 的上一条通知
        $previousNotification = $notifiable->unreadNotifications()
            ->where('data->target->type', $currentNotification->data['target']['type'])
            ->where('data->target->id', $currentNotification->data['target']['id'])
            ->where('id', '<>', $currentNotification->id)
            ->first();

        if ($previousNotification) {
            $compressCount = $previousNotification->data['compress_count'] ?? 1;
            $triggers = $previousNotification->data['triggers'] ?? [$previousNotification->data['trigger']];


            $compressCount += 1;

            // 最多存储三个触发者
            if (count($triggers) < 3) {
                $triggers[] = $currentNotification->data['trigger'];
            }

            $previousNotification->delete();

            $data = $currentNotification->data;
            $data['compress_count'] = $compressCount;
            $data['triggers'] = $triggers;
            unset($data['trigger']);
          
            $currentNotification->data = $data;
            $currentNotification->save();
        }
    }
}
```

压缩后的通知的 data 的 trigger Object 变成了 triggers Array ，并且增加了 `compress_count` 用来记录压缩条数，前端可以通过该字段来判断通知是否被压缩过。

## 补充

- 相应的 API 如通知列表，标记为已读等已经完成，欢迎参考源码。
- 通常会有一个与通知类似**性质**的功能，称为消息，或者说私信/聊天等。该功能依旧可以使用 Notification 来完成，因为其也是由 **行为的触发者 trigger 、行为主体(可能携带 内容) target 、需要通知的用户 notifiable ** 构成。但是从解耦的角度来看，将其单独成一个 「Chat 模块」，且消息提醒依旧使用 「Notification」 来完成会是更好的选择。
- 通常会有一个与通知类似**结构**的功能，称为用户动态，或者说用户日志。上文中有提到该功能，后续会完成该功能。
- 未读消息数在 users 表中冗余 `unread_notification_count` ，而不是进行实时计数。

## 相关

- Api document [postman](https://documenter.getpostman.com/view/1500624/S17nTVKL)
- 线上调试工具 [telescope](http://community.eienao.com/telescope)
- 本节源码 [weiwenhao/community-api](https://github.com/weiwenhao/community-api/tree/0.4.0-alpha)
- Api Tool [weiwenhao/tree-ql](https://github.com/weiwenhao/tree-ql)
- 上一节 [编写具有描述性的 RESTful API (三): 用户行为](https://learnku.com/articles/25824)

