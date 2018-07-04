---
title: 解决Chrome6.2以上Preview无法渲染js问题
date: 2017-12-27 01:31:19
tags:
---


<!-- more -->
## 问题描述
![](http://omjq5ny0e.bkt.clouddn.com/17-12-23/6366700.jpg)
既该界面无法渲染dump()或者dd()
> chrom版本: 63.0.3239.84
> 系统版本:  os10.12

## 解决办法

当响应的状态码为400 or 500系列时可以使Preview渲染

```php
http_response_code(500); // or 400系列状态码
dd(request());
```
![](http://omjq5ny0e.bkt.clouddn.com/17-12-23/14918492.jpg)

寻求方便的话,可以将上面的代码添加到 live templates中,或者添加一个helper function
```
function ddd(...$args){
    http_response_code(500);
    call_user_func_array('dd', $args);
}
```

## 补充
如果服务端响应的数据过大时Preview可能会出现 `failed to load response data` 这样的错误.
并没有找到确切的解决办法,可以尝试变更http响应的状态码为400解决该问题.

参考链接: [https://github.com/symfony/symfony/issues/24688](https://github.com/symfony/symfony/issues/24688)



