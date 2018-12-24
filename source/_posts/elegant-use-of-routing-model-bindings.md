---
title: 简述 container —— 反射机制实现
date: 2018-01-27 17:00:03
tags:
---


<!-- more -->


## 前言

> 使用依赖注入实现控制反转来降低代码的耦合性。

依赖注入 DI，控制反转 IOC，解耦等等等 大家探索前进，终走到了这一步,终于让代码灵活、可扩展、低耦合、高内聚，终于可以开心的coding了!!

好，现在我要去数据库中查询用户id为1的用户名。

```php

new User(
    new \Database\MysqlDrive(new \Database\Connection),
    new Builder(),
    new Relations(),
    new Event(new Listener()),
    new Auth(),
    ...
)

stop？我好像有点写不完，我只想获取一个用户名而已！！！
```


当我使用依赖注入后，所有的依赖都由调用者来处理， 调用者可以控制User的行为，但却要做更多的事情。

但我不想多做一丝事情，我希望更加简单，简单到连依赖也能自动处理，解析，自动注入该有多好呀！

> 题外话：我所理解的最为灵活的php开发架构是，php作为一个中转，将前端和mysql连接起来。这样我们就可以灵活到一套代码解决所有所有的问题了。
> 例如`http://web.dev/users/1?fields=id,username&include=posts:limit(1|5):order(created_at|desc)`


## 示例代码

我现在有一个Foo类，Foo类中一个`allBarName()`方法.并且依赖于BarA~BarD这4个类的name()方法. 如下 

```php
<?php

namespace Container\Foo;

use Container\Bar\BarA;
use Container\Bar\BarB;
use Container\Bar\BarC;
use Container\Bar\BarD;

class Foo
{
    private $barA;
    private $barB;
    private $barC;
    private $barD;

    public function __construct(BarA $barA, BarB $barB, BarC $barC, BarD $barD)
    {

        $this->barA = $barA;
        $this->barB = $barB;
        $this->barC = $barC;
        $this->barD = $barD;
    }

    public function allBarName()
    {
        return $this->barA->name().'/'.$this->barB->name().'/'.$this->barC->name().'/'.$this->barD->name();
    }
}

```

BarA示例， BarB，BarC，BarD相同

```php
<?php

namespace Container\Bar;

class BarA
{
    public function name()
    {
        return '彼得·帕克';
    }
}

```

现在用依赖注入的方式来，调用`allBarName()`方法。

```php
$foo = new \Container\Foo\Foo(
    new \Container\Bar\BarA(),
    new \Container\Bar\BarB(),
    new \Container\Bar\BarC(),
    new \Container\Bar\BarD()
);

echo $foo->allBarName(); // 彼得·帕克/托尼·屎大颗/巴里·艾伦/布鲁斯·韦恩

```
上面的code已经是低耦合的了。
但懒惰的我并不想去手动注入Foo的所有依赖，我希望能有一个Container来帮我解析Foo的所有依赖，然后再把实例返回给我。

那就来吧，实现一个简单的Container

```php
<?php

namespace Container;

class Container
{
    public static function make($class)
    {
        $reflection = new \ReflectionClass($class);

        $constructor = $reflection->getConstructor();
        $parameters = $constructor->getParameters();

        $instances = [];

        foreach ($parameters as $parameter) {
            $className =  $parameter->getClass()->getName();
            $instances[] = new $className;
        }

        //... 将数组拆封成参数传递给方法, 类似于 call_user_func_array()
        return new $class(...$instances);
    }
}
```

这是一个简单到不能再简单的依赖解析容器了，但我想它能满足我的需求。

再次调用allBarName()方法

```php
$foo = \Container\Container::make(\Container\Foo\Foo::class);
echo $foo->allBarName(); // 彼得·帕克/托尼·屎大颗/巴里·艾伦/布鲁斯·韦恩
```
oh yeah！成功了


## 结语

我想我已经表达了我的想法，我还没有能力也不是很想去实现一个强大灵活的container，能够知道为什么需要container我已经很满足了。

- code demo [https://github.com/weiwenhao/container](https://github.com/weiwenhao/container)


