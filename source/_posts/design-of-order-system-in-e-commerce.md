---
title: E-commerce 中订单系统的设计
date: 2018-12-22 18:28:10
tags: e-commerce order laravel
---


<!-- more -->

## 数据库设计

#### Order

订单系统的核心表自然是 **orders**系列表,laravel的迁移文件如下

```php
Schema::create('orders', function (Blueprint $table) {
    $table->increments('id');
    $table->string('number')->nullable()->comment('订单号');
    $table->unsignedInteger('address_id')->nullable()->comment('订单地址');
    $table->unsignedInteger('user_id')->index()->comment('用户id');
    $table->integer('items_total')->default(0)->comment('order每一个item的total的和 unit/分');
    $table->integer('adjustments_total')->default(0)->comment('调整金额 unit/分');
    $table->integer('total')->default(0)->comment('需支付金额 unit/分');

    $table->string('local_code')->comment('语言编号');
    $table->string('currency_code')->comment('货币编号');

    $table->string('state')->comment('主状态 checkout/new/cancelled/fulfilled');
    $table->string('payment_state')->comment('支付状态 checkout/awaiting_payment/partially_paid/cancelled/paid/partially_refunded/refunded');
    $table->string('shipment_state')->comment('运输状态 checkout/ready/cancelled/partially_shipped/shipped');

    $table->ipAddress('user_ip')->comment('用户ip ip2long后的结果');

    $table->timestamp('paid_at')->nullable()->comment('支付时间');
    $table->timestamp('confirmed_at')->nullable()->comment('确认订单时间');
    $table->timestamp('reviewed_at')->nullable()->comment('评论时间');
    $table->timestamp('fulfilled_at')->nullable()->comment('订单完成时间');

    $table->json('rest')->nullable()->comment('非核心字段冗余');

    $table->timestamps();
});
```

接下来是**order_items**表,用于记录order的item

```php
Schema::create('order_items', function (Blueprint $table) {
    $table->increments('id');
    $table->unsignedInteger('order_id')->index()->comment('外键');
    $table->unsignedInteger('variant_id')->comment('variant是国外的称呼,国内通常称为sku. 既库存最小单位');
    $table->unsignedInteger('product_id')->comment('冗余字段');
    $table->unsignedInteger('quantity')->comment('购买数量');

    // adjustment calculate
    $table->integer('units_total')->default(0)->comment('item中每一个unit的和. 单位/分');
    $table->integer('adjustments_total')->default(0);
    $table->integer('total')->default(0)->comment('units_total + adjustments_total');
    $table->integer('unit_price')->default(0)->comment('variant单价,冗余字段');

    $table->json('rest')->nullable()->comment('非核心字段冗余');
    $table->timestamps();
});
```

> 做过海外电商或者亚马逊的朋友应该对variant(变体)不陌生. 国内称为sku. 每一个商品都会有多个变体

接下来是**order_item_units** 表

```php
Schema::create('order_item_units', function (Blueprint $table) {
    $table->increments('id');
    $table->unsignedInteger('item_id')->index();
    $table->unsignedInteger('shipment_id')->comment();
    $table->integer('adjustments_total')->default(0);
    $table->timestamps();
});
```

对于用户购买的每一件实体,我们都需要谨慎的做一条记录,其会涉及到运输/促销/退货等问题, 例如variantA我们购买了三件,那么我们就需要为这三件相同的变体分别创建三条记录.

上面三张表的关系从上往下 一个order会有多个item,一个item根据quantity的值,会有对应数量的unit. 

>  order和order_item表大家应该都知道. 
>
> order_item_units表可能有些同学第一次知道,但是其是必要存在的

*tip: 所有的价格字段都使用分为单位存储,从而避免小数在计算机系统中存在的一些问题*

可以消化梳理一下上面的三张订单系统核心表,然后再介绍一下其他相关表的设计. 数据库的设计应该是灵活的,可以根据实际的需求任意添加和修改字段

#### Adjustment

上面三张表都出现了adjustment_total字段,可能会有些疑惑.

如果我们每个变体的价格是10元,那我买三个这件变体则需要30元,但是实际支付的金额往往都不是30元.,会有各种各样的情况影响我们最终支付的价格.

比如运费+5元,促销折扣 -8元,税收+3元,退还服务 +0.5元,最后实际需要支付 35.5元.  为什么30元的金额最后却支付了35.5元?

我们不能凭空蹦出个35.5元,影响商品实际支付金额的每一个因素都是至关重要,我们需要负责任的记录下来.这便是adjustment表的来源.

首先看看迁移文件

```php
Schema::create('adjustments', function (Blueprint $table) {
    $table->increments('id');

    $table->unsignedInteger('order_id')->nullable();
    $table->unsignedInteger('order_item_id')->nullable();
    $table->unsignedInteger('order_item_unit_id')->nullable();

    $table->string('type')->comment('调整的类型 shipping/promotion/tax等等');

    $table->string('label')->comment('结合type决定');

    $table->string('origin_code')->comment('结合label决定');

    $table->bool('included')->comment('是否会影响最终订单需要支付的价格')
        $table->integer('amount');
    $table->timestamps();

    $table->index('order_id');
    $table->index('order_item_id');
    $table->index('order_item_unit_id');
});
```

调整对订单价格的影响分为三种类型, 分别是 影响整个order, 影响order_item(较少预见),影响order_item_units. 

> included字段 用来判断本条adjustment记录,是否会影响消费者最终需要支付的金额
>
> 大部分的adjustment都会影响最终结算的价格, 小部分如商品税,通常已经计算在了商品的单价中, 不会影响消费者最终需要支付的金额.但是在开具发票时 却需要展示,因为我们做必要的记录

举个例子, 假设我们一笔订单的运费是5元,那么会有这样一条adjustment记录

```json
{
    id: 1,
    order_id: 1,
    order_item_id: null,
    order_item_unit_id: null,
    amount: 500,
    type: 'shipping',
    label: 'UPS',
    origin_code: null,
    included: 1,
}
```

假设我们消费者在一个订单中购买了三条1.5米数据线,并使用了一张8元的代金券,那么会有这样三条adjustment记录

```json
[
    {
        id: 2,
        order_id: null,
        order_item_id: null,
        order_item_unit_id: 1,
        amount: -267,
        type: 'promotion',
        label: '8元代金券',
        origin_code: 'KSDI12K2', // 代金券code
        included: 1
    },
    {
        id: 2,
        order_id: null,
        order_item_id: null,
        order_item_unit_id: 2,
        amount: -267,
        type: 'promotion',
        label: '8元代金券',
        origin_code: 'KSDI12K2', // 代金券code
        included: 1
    },
    {
        id: 2,
        order_id: null,
        order_item_id: null,
        order_item_unit_id: 3,
        amount: -266,
        type: 'promotion',
        label: '8元代金券',
        origin_code: 'KSDI12K2', // 代金券code
        included: 1
    },
]
```



实际上对于大部分的促销需求 我们都应该将促销的折扣金额均分到每一个unit中.

这样设计的一个好处是,当消费者退调用其中一根数据线时,我们可以很清楚的计算出应该退多少金额给消费者. 既 `单价 + order_item_unit.adjustment`

实际上清楚的记录每一笔影响最终支付金额的adjustment,无论对消费者还是对供应商来说都是负责的做法.

>  运费为什么不需要分摊到unit?
>
> 运费对于一笔订单来说,是固定的外部消费(由快递公司获利),退款时商家并不需要为运费负责, 只需要退还商品的等额价值即可
>
> 更加白话的说法就是 你在淘宝买了一个商品20元,运费10元, 你觉得商品不好想要退货(不考虑寄回的运费), 商家需要退你30元吗?

#### Shipment/Payment

shipment为订单的运输信息存储,payment为支付信息存储.先来看看迁移文件

```php
Schema::create('shipments', function (Blueprint $table) {
    $table->increments('id');
    $table->unsignedInteger('method_id')->comment('运输方式 外键');
    $table->unsignedInteger('order_id')->comment('订单 外键');
    $table->string('state')->comment('运输状态');
    $table->string('tracking_number')->nullable()->comment('订单号码');
    $table->timestamps();

    $table->index('order_id');
});
```

```php
Schema::create('payments', function (Blueprint $table) {
    $table->increments('id');
    $table->unsignedInteger('method_id')->comment('支付方式');
    $table->unsignedInteger('order_id');7
    $table->string('currency_code', 3)->comment('冗余 货币编码');
    $table->unsignedInteger('amount')->default(0)->comment('支付金额');
    $table->string('state');
    $table->text('details')->nullable();
    $table->timestamps();

    $table->index('order_id');
});
```

上面在order_item_units表中存在一个shipment_id 就对应这里的shipment表. shipment和order_item_units之间是**一对多**的关系,订单中的每一个实体都可以被分别运输,例如京东购物时经常会见到这种情况. 

**一条shipment/payment 会和一条实际存在的货运记录/支付记录(退款记录) 挂钩.**

上面就是订单系统的核心表了,对于后端来说,数据库就已经可以反映出整个系统的设计了.

接下来抽出一些细节进行详细的介绍

## 业务设计

#### 状态的设计

相信很多小伙伴在做订单系统时会被各种状态 待确认,待支付,待发货,已发货,关闭订单 等等弄的晕头转向,今天我们就来梳理一下订单系统中的各种状态

如果各种状态只在order表使用一个state字段来记录显得有些力不从心,因此推荐使用三个字段,它们分别是 **state,shipment_state,payment_state.** 来分别记录在订单中我们或者消费者最关心的三种状态. 

先来分别看看三个state的状态转移图

**order.state↓**

![](http://asset.eienao.com/18-12-22/78777487.jpg)

这是一笔订单的几个最基本的几个状态. 

先讲一讲初始状态,既 checkout, 这与订单在什么时候创建有关系,当消费者在购物车点击结账时,就创建了一个订单,用于本次结账, 因此订单的初始状态为checkout

> 结账也就是所谓的确认订单页,在该页面中,消费者可以选择优惠券,选择地址等操作

处于该状态的订单对于后台管理系统/用户个人中心都是不可见的,且checkout类型订单的创建,也不会是库存有任何的变化

当用户在结账界面操作完成后需要用户点击确认订单. 既行为 **confirm**的触发,使订单的状态从checkout转换成了new. 此时的订单无论是对于消费者/运营人员/仓储系统来说,都是真实存在且有效的. 且响应的购物车记录也被清空. 对于一笔状态为new的订单,消费者可以对其行使付款的权利. 

**order.payment_state↓**

![](http://asset.eienao.com/18-12-22/4209529.jpg)

payment的初始状态为checkout与上述一致.

当消费者触发confirm后, 我们就可以触发request_payment行为,将订单的付款状态转换为 await_payment, 且将消费者引导到支付界面, 当消费者支付成功后,在支付成功的回调中,触发pay行为,将支付状态转换为paid.

关于退款的状态如上图所示,需要注意的是,对于退款,会出现只需要退订单中的部分商品的情况,因此加入了 partially_refunded(部分退款的状态). 

**order.shipment_state↓**

![](http://asset.eienao.com/18-12-22/15354759.jpg)

当消费者confirm后, 我们同时也需要调用响应的request_shipment,将我们的运输状态设置为一个ready状态,此时库存已经锁定.

> 关于仓库具体的备货时机 是在用户确认订单之后,还是等用户支付完成之后,需要根据实际的产品需求确定.
>
> 上面的状态图属于前者,当消费者确认订单后,便锁定了库存,并开始了备货阶段.如果是后一种情况可以将checkout修改为pending,等待消费者付款完成后再将状态转移到ready



**对于上面繁杂的状态转换,可以手动处理,也可以选择使用[state-machine](https://github.com/weiwenhao/state-machine) 进行处理**

#### 订单价格的计算

单价作为一件商品的固有属性,不会受到运输/促销折扣等等因素的影响. 当商家对一个价值100元商品进行一个30%的折扣时,消费者只需要用70元的价格买入, 但实际上商品的单价依旧是100元.

当一笔订单不存在任何的adjustment时,我们可以很容易的计算出订单的实际支付价格, 只需要把各个order_item的unit_price * quantity 相加起来即可

但是有了adjustment参与之后,我们必须自下往上的计算. 下面的例子是在laravel项目且使用了上述的数据库设计后的一个计算方法.

```php
public function calculator(Order $order)
{
    $items = $order->items;
    $items->load('adjustments', 'units.adjustments');
    $order->load('adjustments');

    $items->each(function ($item) {
        $item->units->each(function ($unit) {
            $unit->adjustments_total = $unit->adjustments->sum('amount');
        });

        $item->units()->saveMany($item->units);

        $item->adjustments_total = $item->adjustments->sum('amount');

        $item->units_total = $item->quantity * $item->unit_price + $item->units->sum('adjustments_total');

        $item->total = $item->units_total + $item->adjustments_total;
    });

    $order->items()->saveMany($items);

    $order->adjustments_total = $order->adjustments->sum('amount');
    $order->items_total = $order->items->sum('total');
    $order->total = $order->items_total + $order->adjustments_total;
    $order->save();
}
```





#### 补充

1. 当订单创建的同时(结账阶段)就分别创建了一条payment/和shipment记录.在payment和shipment中分别记录了用户选择的支付方式与运输方式.

   在电商系统中,通常会有多种多样的支付方式和运输方式.

   但是在实际的业务编写时,业务层并不希望关心和处理繁杂的支付与运输方式,此时支付网关和运输网关便应运而生,其对业务层隐藏了繁杂的细节,而暴露出了统一的api接口.

   支付网关如提供商业服务的 ping++,当然也有一些开源项目对这方面有所支持. 如 [yansongda/pay](https://github.com/yansongda/pay) , [Payum/Payum](https://github.com/Payum/Payum)等等

2. 对于确认了但超过一定时间没有付款的订单,我们可以选择主动关闭该订单. 将order.state/order.payment_state/order.shipment_state 设置为cancelled,并对库存进行归还等系列操作

下一篇将会介绍促销系统的设计与实现,本篇的主要目的是介绍订单系统的相关设计,为下一篇做一个铺垫. 

由于篇幅有限并没有过多的细节,有疑问或者不妥的地方欢迎留言.
