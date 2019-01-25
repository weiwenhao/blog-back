---
title: E-commerce 中促销系统的设计
date: 2018-12-26 16:04:20
tags: e-commerce promotion laravel
---


在电商平台中,促销是必不可少的营销手段,尤其在国内 各种玩法层出不穷,最开始的满减/秒杀 到优惠卷 再到 拼团/砍价等等

一个良好的促销系统应该具备易于扩展,易于统计促销效果等特点,在遇到秒杀类促销时还需要做到可扩容,抗并发(本次不考虑秒杀系统的设计)等等.  废话说完了,进入正题吧

<!-- more -->

## 概览

![](http://asset.eienao.com/18-12-25/96451684.jpg)

对各种促销行为进行分析,会发现本质上是由两个部分和一个作用域组成.

**促销的核心作用域既订单**.因此我在上一篇文章中介绍了电商中订单系统的设计 [E-commerce 中订单系统的设计](https://laravel-china.org/articles/21461)

两个部分既上图中的rule和action部分. 

**rule描述了促销限制**,既订单需要满足那些条件才能参与某个促销.常见的促销限制有 订单金额/购买时间/购买数量/收货地址/支付方式/用户类型/购买人数 等等.

**action描述了给予订单哪些优惠策略** 如折扣/直减/免运费/返现/赠品 等等.

这样设计最大好处是 **rule与action相互独立且高度抽象**, 运营人员与开发人员可以**自由组合rule和action来达到最大灵活性与可扩展性**

## 数据库设计

#### Promotion

```php
Schema::create('promotions', function (Blueprint $table) {
    $table->increments('id');
    $table->string('code');

    $table->string('name')->nullable();
    $table->string('description')->nullable();
    $table->string('cover')->nullable()->comment('促销封面');
    $table->string('asset_url')->nullable()->comment('促销详情链接')

    $table->integer('position')->default(0)->comment('权重');
    $table->string('type')->comment('优惠卷/满减促销/品牌促销/秒杀/拼团/通用.');

    $table->json('config')->nullable()->comment('配置');

    $table->timestamp('began_at')->nullable()->comment('促销开始时间');
    $table->timestamp('ended_at')->nullable()->comment('促销结束时间');
    $table->timestamps();
    $table->softDeletes();
});
```

为了实现良好的促销效果统计行为,所有的促销行为都应该对应promotion表中的一条记录.

#### Rule

```php
Schema::create('promotion_rules', function (Blueprint $table) {
    $table->increments('id');
    $table->unsignedInteger('promotion_id');
    $table->string('type');
    $table->json('config')->nullable();
    $table->timestamps();

    $table->index('promotion_id');
});
```

常见的rule type有

- 订单总额 order_total
- 订单中促销项目总额 promotion_items_total
- 第N笔订单  nth_order
- 所属分类 has_category
- 消费者用户组 customer_group  (白金会员组/钻石会员组 等等)
- 购买数量  item_quantity 
- 等等

#### Action

```php
 Schema::create('promotion_actions', function (Blueprint $table) {
     $table->increments('id');
     $table->unsignedInteger('promotion_id');
     $table->string('type');
     $table->json('config')->nullable();
     $table->timestamps();

     $table->index('promotion_id');
 });
```

常见的action type有

- 订单固定折扣 order_fixed_discount
- 订单百分比折扣 order_percentage_discount
- 订单中促销项目固定折扣 promotion_items_fixed_discount
- 订单中促销项目阶梯式折扣 promotion_items_ladder_discount
- 赠送积分 present_integral
- 运费百分比折扣 shipping_percentage_discount
-  等等



> json类型的config字段的灵活应用是促销系统灵活的另一个主要原因
>
> 关于json字段的使用细项,及索引方式 可以参考  [MySQL 中 JSON 字段的使用技巧](https://laravel-china.org/articles/20985)



#### PromotionVariant

在常见的电商平台中,一个促销活动通常不会涉及所有的商品, 尤其是类似淘宝这种B2C模式的平台,促销通常是以商家报名的形式展开的. 因此我们会有一个表来记录 有哪些变体(variant)参与了本次促销.

> 变体(variant)即sku, 下文将统称为变体.
>
> 另外不以product作为参与促销的最小单位, 是为了进行更细颗粒度的控制.

一个促销可以有多个变体参与,一个变体可以同时参与多个促销. 因此 promotion_variants 实际上是promotions表和variants表中间的一张中间表, 并且这张中间表携带了其他信息, 来看看迁移文件

```php
Schema::create('promotion_variants', function (Blueprint $table) {
    $table->increments('id');
    
    $table->unsignedInteger('variant_id')->index();
    $table->unsignedInteger('promotion_id')->index();

    $table->decimal('discount_rate')->nullable()->comment('折扣率, 值为0.3表示打7折');
    $table->unsignedInteger('stock')->nullable()->comment('促销库存');
    $table->unsignedInteger('sold')->default(0)->comment('销售数量');
    $table->unsignedInteger('quantity_limit')->nullable()->comment('购买数量限制');
	$table->boolean('enabled')->default(1)->comment('启用');
    
    // 冗余
    $table->unsignedInteger('product_id');
    $table->string('promotion_type')->comment('冗余promotion表type');
    $table->json('rest')->nullable()->comment('冗余');

    $table->timestamps();
});
```

上面便是促销系统的核心表,数据库字段可以按照实际需求进行增减和修改,特殊促销可自行添加相关表, 如优惠卷促销的coupons表, 拼团的groups表, 报名促销的promotion_sign_up表等等

## 业务设计

#### 流程设计

以一次圣诞节满减促销为例,第一步的工作是创建promotion和相应的rules和actions. 我们首先会有这样3条记录

```json
// promotion
{
    id: 1,
    code: '2018-christmas',
    name: '圣诞节满减大促',
    type: 'full_discount',
    description: '促销商品满100减10元',
    cover: null,
    asset_url: null,
    rest: null,
    config: null,
    position: 0,
    began_at: '2018-12-25 00:00:00',
    ended_at: '2018-12-26 00:00:00'
}

// rule
{
    'id': 1,
    'promotion_id': 1,
    'type': 'promotion_items_total', // 订单中促销项总额
    'config': {
    	'amount' => 10000, // unit/分 
	}
}

// action
{
    'id': 1,
    'promotion_id': 1,
    'type': 'promotion_items_fixed_discount', // 订单中促销项 固定折扣
    'config': {
    	'amount' => 1000, // unit/分 
	}
}
```

当促销创建完成后,下一步就是确定本次促销的变体了.

对于自营网站,由网站运营创建促销,挑选变体并添加到promotion_variants表中.对于B2C平台,由网站运营创建促销,商家选择变体并报名参与本次促销,运营审核后将其添加到相应的promotion_variants表中.

当促销的变体确定后. 对于有需要的促销,可以为促销设计聚合页面/详情页/宣传页/推广页,然后将相应的链接和封面添加到promotion.asset_url和promotion.cover中保存即可. 

#### 代码逻辑

订单对促销的判断的逻辑的laravel伪代码

```php
// 获取平台所有有效的促销
$promotions = Promotion::active()->get();

// 通过rule过滤promotion
$promotions = $promotions->filter(function ($promotion) {
    $rules = $promotion->rules
    $order = $this->getOrder();
    
    // 判定订单是否满足所有rule,当存在一条rule不被订单所满足,应返回false,被过滤器过滤掉
        
    return true;
});

// 为订单应用action.
$promotion->each(function ($promotion) {
    $actions = $promotion->actions;
    $order = $this->getOrder();
    
    // 将actions逐条应用于订单
})

```

> **特别注意:** 对订单应用actions并不意味着直接修改订单中的商品单价或支付总额等. 而应有条理的记录影响订单支付金额的行为和原因. 既使用上一篇中提到的adjustment来记录  [E-commerce 中订单系统的设计](https://laravel-china.org/articles/21461)



关于action和rule的代码逻辑可以先来看两个interface

```php
<?php

namespace Promotion\Constructs;


interface Checker
{
    public function isEligible(array $configuration): bool;
}
```

```php
<?php

namespace Promotion\Constructs;


interface Action
{
    public function execute(array $configuration);
}

```

每一条rule的设计都要实现上面的 Checker接口,每一条action都要实现上面的Action接口. 

以上面的圣诞满减促销的rule和action为例子,来看看具体的实现

```php
<?php

namespace Promotion\Checker;

/**
 * 有很多的通用方法 如getOrder,getPromotionOrderItems等. 
 * 因此我创建了一个基类checker来实现interface和通用方法
 */
class PromotionItemsTotalChecker extends Checker
{
    public function isEligible(array $configuration): bool
    {
        return $this->getPromotionOrderItemsTotal() >= $configuration['amount'];
    }
}
```

> 需要注意一点,一笔订单中可能存在许多变体,但通常情况是只有部分变体参加了圣诞大促.因此我们计算购物总额时应该使用order中参与了圣诞促销items




```php
<?php

namespace Promotion\Actions;

use Promotion\Helpers\CreateAdjustment;
use Promotion\Helpers\Distribute;

class PromotionItemsFixedDiscountAction extends Action
{
    use Distribute, CreateAdjustment;

    public function execute(array $configuration)
    {
        // 满减的金额
        $amount = $configuration['amount'];

        if ($amount === 0) {
            return false;
        }

        // 格式校验, amount如果小于订单金额时,则使用订单金额作为优惠amount
        $amount = -1 * min($this->getPromotionOrderItemsTotal(), $amount);

        if ($amount === 0) {
            return false;
        }

        $items = $this->getPromotionOrderItems();

        $itemsTotals = [];
        
        foreach ($items as $item) {
            $itemsTotals[] = $item->total;
        }

        // 促销金额等比例分配.
        $splitAmount = $this->distributeAmountOfItem($itemsTotals, $reduceAmount);

        // 创建adjustments
        $this->createUnitsAdjustment($items, $this->getPromotion(), $splitAmount);

    }
}

```

> **本文的主要目的是提供思路与想法, 因此没有太过具体完整的代码.**
>
> 未来如果有机会的话会设计一些促销系统扩展等提供参考.



上面便是一个促销系统的流程思路,下面多提供一些demo供参考



#### 优惠卷

已一张10元代金卷为例,我们会有这样两条记录

```json
// promotion
{
    id: 1,
    code: '10-cash',
    name: '10元代金券',
    type: 'coupon',
    description: '全场可用',
    cover: null,
    asset_url: null,
    config: {
        type: 'cash',
        reduce_amount: 1000, // 冗余自下面action中的config中的amount
        stock: 10000, // 库存数量
        sold: 0, // 已经领取的数量
        catch_limit: 1, // 领取限制
        date_type: 'fix_term', // 固定期限
        fix_term: 30, // 自领取日内30天有效,
        
        // date_type: 'fix_time_range', 固定时间段
        // began_at: '2018-12-23 00:00:00',
        // ended_at: '2018-12-25 00:00:00',
    },
    position: 0,
    began_at: '2018-12-25 00:00:00',
    ended_at: '2018-12-26 00:00:00'
}

// action
{
    'id': 1,
    'promotion_id': 1,
    'type': 'order_fixed_discount', // 订单中促销项 固定折扣
    'config':{
       'amount' => 1000, // unit/分  
    }
}
```

代金券通常没有使用限制,因此不需要rule.

代金券通常是全场可用, 因此action我们使用 order_fixed_discount,而不是promotion_items_fixed_discount.

对于config中的配置适用于各种优惠卷,如满减卷,运费卷等等. 

对于满减卷的配置只要再为这笔促销添加一个类型为`promotion_items_total` (部分变体满减)或者`order_total`(全场满减) 的rule即可

优惠卷促销通常要创建一个 coupons表来存储用户领取的优惠卷及使用情况等

优惠卷促销本质上是将传统促销以卷的形式体现了出来,既圣诞满减促销 => 圣诞满减卷的转换.

#### 秒杀/直减/聚划算

直减类型促销通常是已变体为单位进行高折扣的促销行为,秒杀具体要折扣多少通常不是统一设定的,不同的变体会有不同的折扣率,所以可能会有这样两条记录

```json
// promotion
{
    id: 1,
    code: 'unit-discount-1290',
    name: '1290期直减',
    type: 'unit_discount',
    description: null,
    cover: null,
    asset_url: null,
    config: null,
    position: 0,
    began_at: '2018-12-25 00:00:00',
    ended_at: '2018-12-26 00:00:00'
}

// promotion_variant 
{
    'id': 1,
    'variant_id': 1,
    'prootion_id': 1,
    'discount_rate': 0.35,
    'stock': 100, // 秒杀库存
    'sold': 0,
    'quantity_limit': 1, // 限购
    'enabled': 1,
    'product_id': 1,
    'promotion_type': 'unit_discount',
    'rest': {
        variant_name: 'xxx', // 秒杀期间变体名称
        image: 'xxx', // 秒杀期间变体图片
    }
}
```

promotion_variant 由运营添加或者供应商报名得到.直减并没有相应的rule/action组合而来, 属于特殊促销.

但是在代码逻辑中依旧可以提现出这种特殊的rule和action

既`UnitDiscountChecker`来判定订单是否可以参与本次秒杀促销,

通过`UnitDicountAction`来记录相应的PromotionOrderItems的折扣信息,既下面的伪代码

```php
// rule验证阶段
if ($promotion->type === 'unit_discount') {
   return (new UnitDiscountChecker)->isEligible()
}

// 应用action阶段
if ($promotion->type === 'unit_discount') {
    (new UnitDiscountAction)->execute()
}
```



#### 阶梯式满减

阶梯式满减属于传统满减促销的一个变种.下面是一个 满100 - 10,满150 - 20,满200 - 30的阶梯式满减的action记录.

```json
// action
{
    'id': 1,
    'promotion_id': 1,
    'type': 'promotion_items_ladder_discount',
    'config': {
        "ladder": [
            {
                "least_amount": 10000,
                "reduce_amount": 1000
            }, {
                "least_amount": 15000, 
                "reduce_amount": 2000
            }, {
                "least_amount": 20000, 
                "reduce_amount": 3000
            }
        ]
	}
}
```

具体的ladder应该由运营人员后台设定,实际上对于每一种action和rule的type,在后台管理界面中都应该设置其相应的表单交互



## 结

如果你有疑惑或者更多的想法欢迎留言.
