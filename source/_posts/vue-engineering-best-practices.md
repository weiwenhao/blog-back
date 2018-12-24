---
title: Vue 工程化最佳实践
date: 2018-12-14 11:49:27
tags: vue
---

<!-- more -->

### 目录结构

总览

![](http://asset.eienao.com/20181214113352.png)

- api目录用于存放 api请求,文件名与模型名称基本一致,文件名使用小驼峰, 方法名称与后端restful控制器一致.
  - ![](http://asset.eienao.com/20181214113459.png)

- enums 目录存放 常量, 与后端的常量目录对应
  - ![](http://asset.eienao.com/20181214113531.png)

- icons目录用于存放图标, element-ui提供的图标实在是太少啦.所以我通常会使用 阿里的iconfont
- lang目录存放多语言
- layouts目录存放布局
  - ![](http://asset.eienao.com/20181214114528.png)
  - 上面展示的是一个后台系统, empty为一个空布局.用于登录页面, 其他页面则使用default布局. 布局不需要过多介绍,写过laravel blade都很熟悉了.这里的布局需要和vue-router配合使用
- mixins 类似php的trait, 但是它更强大, 完整贴合vue组件的生命周期
- plugins 目录存放插件配置, 比如 axios,vue-lazy等 (这是从nuxt中学到的概念)
  - ![](http://asset.eienao.com/20181214114547.png)

- router目录存放与 前端路由相关的配置,总体来说类似于laravel的api层
- store 目录即vuex的目录, 类似于前端的model. 其文件与后端model相匹配,采用小驼峰命名
  - ![](http://asset.eienao.com/20181214114609.png)

- utils 目录存放辅助函数
- views 为业务视图层,相信后端同学也很熟悉.其由vue-router直接调度
- main.js 为app的入口, 类似于后端的index.php
- components 目录, 存放组件.通常是一些可复用的组件会单独存放在该目录

总体来说, 已后端的mvc思想来看现代的前端项目是非常的自然的. 后端的model对应前端的store, 后端的router对应前端的router,后端的controller + views 对应前端的views.

### 基础规范

 就目前来说 vue项目很少用到 class, 因此 .js文件通常都是一个 module, 所以文件名使用小驼峰的形式命名. 如果有类文件,则类文件使用大驼峰的形式命名.

.vue文件 可以使用 中划线和大驼峰两种命名方式, 参考了element/iview/nuxt项目之后, 推荐统一使用中划线命名.

所有的文件夹名称统一使用中划线命名

> 引入vue组件时文件时需要转换成大驼峰` import 'TestTest' from '@/components/test-test' `
>
> 在template 使用时依旧使用中划线
>
> `<test-test />`

其他规范如变量命名和使用规范 使用eslint的standard 即可很好的解决.

前端存在很多的事件 如change/input/upload/sumit等等,相应的处理推荐使用 handle + 事件名称, 如`handleChange`

### 生命周期

vue-router 解析当前用户键入的 url, 然后匹配合适的视图组件加载. 

着重介绍一下 我对views目录下的视图组件的理解,已修改地址为例

![](http://asset.eienao.com/20181214114635.png)

script部分既控制器部分,其请求数据, 然后注入到view 中, 就像后端的mvc一样.只不过vue将 vc 其写入到了一个文件中.这样理解对于写过后端的同学显得更加的自然

##### 控制器如何获取数据?

在过去的vue项目中,我们可能会见到这样的写法

```javascript
// ... views/address/edit.vue
created () {
    axios.get('/addresses/1')
        .then((response) => {
             this.list = resposne.data              
        })
}

//...
```

这种写法无异于后端在控制器中写sql语句一样,在工程化实践中不推荐这么做,后端通过model来获取数据会更加的优雅自然.在vue项目中, model既vuex,因此推荐这么做

> 如果你对我说的东西一脸懵逼, 那么你可以看一下 vuex的文档. 我现在做的就是用后端熟悉的概念,来描述前端项目的最佳实践

```javascript
// ... views/address/edit.vue 控制器+视图
computed: {
    address: () => this.$store.address.itemBy[1] // 从store模型中取出我们想要数据
}
// ...

```



```javascript
// ... store/modules/address.js  数据源
export default {
    state: {
        itemBy: {}
    },
    actions: {
        ...
    },
    mutations: {
        ...
    }
}
// ...
```



##### store中的数据从哪里来?

数据当然是从后端的数据库中获取,我们不能让前端直接访问我们的数据库,因此我们会提供api让前端访问.store中存在一个发起api请求的地方,既 action.

```javascript
// ... store/modules/address.js  模型
export default {
    state: {
        itemBy: {}
    },
    actions: {
        async fetchItem({ commit, state }, { id }) {
             // 对axios和api进行了简单的封装,使api请求更加语义化
            cosnt { data } = await address.show(id)
            
            // action只能通过提交commit来修改state,具体原因请查看vuex文档 (其实我也忘了为啥 (╯﹏╰))
            commit('SET_ITEM', data) 
        }
    },
    mutations: {
        SET_ITEM: (state, item) => {
            state.itemBy[item.id] = item
        }
    }
}
// ...
```

这样我们的模型中就有数据啦

##### 什么时候去调用`fetchItem`去请求后端呢?

> [https://router.vuejs.org/zh/guide/advanced/data-fetching.html](https://router.vuejs.org/zh/guide/advanced/data-fetching.html) vue-router文档的解答

这里不推荐在vue原始的生命周期中去调用初始化请求,可能会带来 数据还没有获取到,template却已经被渲染.会造成一些数据不存在的异常,推荐在vue-router的生命周期中去请求数据

```javascript
// ... views/address/edit.vue
async beforeRouteEnter (to, from, next) {
    // 等待模型数据加载完毕,才继续进行vue组件的生命周期
    await store.dispatch('fetchItem', to.params.id) 
    
    next()
}
created () {
    // 不推荐在这里调用 fetchItem
}

//...
```

到这里你可能发现,这和你平时写的vue有些不一样, 没有类似 `this.data = response.data` 这种操作. 类似这种操作其实类似赋值操作,或者称为副作用,其引入了时间的概念,使数据的管理变的复杂. 直观的体现就是我们可能会有这种多余的 `if(data) `判断.

当然副作用是难以避免的,但是我们可以统一的管理他们.类似上面的代码就是一套我觉得还不错的方法.从view的角度看, 数据是固有存在存在的,其不需要关心是否是否已经被加载完毕,且store中的不可被view修改,既数据只能单向流动

在store中统一管理数据的另外一个好处就是方便持久化

##### view层如何修改数据源?

上面的描述实际上表达了一种 发布与订阅的模式, 从store到view的数据流是严格单向数据流动.

view层不允许直接修改store中的数据,但是view层却可以通过发送action来影响数据源. 

比如初始化时的dispatch的action,各种event触发的dispatch. 当数据源发生改变时,作为订阅者的view层会非常自然的重新渲染.

>这种设计和父子组件类似,vue中子组件不允许直接修改父组件props到子组件的数据,只能通过向父组件emit event. 在view和store之间,这种设计依然合理.
>
>这也意味着应用中所有的数据都遵循相同的生命周期，这样可以让应用变得更加可预测且容易理解。



![](http://asset.eienao.com/20181214114653.png)

上面的图很好的阐述了这种开发模式.  引自 [https://github.com/sorrycc/blog/issues/1](https://github.com/sorrycc/blog/issues/1)

### view层再深入

view层的script部分,除了充当了传统的controller,起到初始化的作用外,实际上还做了更多的事情. 

先从data部分说起,在view层会有一些状态需要记录, 如 菜单的展开或收起, 弹窗的弹出与关闭. 对于这样的状态的管理,一种做法就是将存储在data部分.

> 也有人将所有的状态 也放在 store中的state维护. 既state分为状态和数据 两种类型.

view层的script更重要的部分,是其到了一个交互反馈的作用, 既类似下面的代码

```vue
<template>
	<button @click="handleSubmit"/>
</template>

<script>
    export default {
        data: {},
        methods: {
            handleSubmit() {
                
            }
        }
    }
</script>
```

关于css部分,由于个人不了解css,也不清楚css的业界规范及在vue上的最佳实践,因此不做过多介绍.



### 总结一下

在前面的介绍中, store和api 目录是和数据挂钩的,当数据库定下来,这一部分也就定了下来.

 views/components 和业务(ui)挂钩,需要等设计稿确定后,这一部分才能确定下来.

> PS设计稿的图层通常就是组件的拆分规范 ?
> 使用vuex存储数据的另一个好处就是可以无缝的切换到ssr框架nuxt

views和store之间是一种订阅和发布的模式.

#### 两个问题

**store中的state应该如何组织?** 对于api请求,我们经常会看到这样的json数据

```json
// post
{
    id: 1,
    title: xxxx,
    content: xxxx,
    user: {
        id: xxx,
        nickname: xxx,
        avatar: xxx,
    },
    comments: [
        {
            id: xxx,
            user_id: xxx,
            content: xxx,
            user: {
                // ...
            }
        },
        {
            //....
        }
    ]
}

```

上面的数据结构复杂,嵌套深入.  如果我们将他们一股脑的存在 post的state中,会造成数据过于集中,冗余. comments无法独立化更新等等问题.  使得前端 scheme/orm,数据组织的规范化变的迫切需要

但是vue在这方面没有很好的规范和最佳实践. react在这方面比较不错的实践 [https://github.com/paularmstrong/normalizr](https://github.com/paularmstrong/normalizr)

**如何设计良好规范的compoents?** 

组件的设计在业务层非常的重要,在下一篇我会介绍一下我总结出的一些实践
