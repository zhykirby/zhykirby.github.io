---
layout:     post
title:      vue-router学习笔记
subtitle:   
date:       2019-07-05
author:     zhykirby
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - vue 
    - vue-router
---


# 前言

学了vue，一般来说vue-router都是绕不开必学的吧……  
整理下学习vue-router中一些比较重要或者有趣的知识。


# 什么叫路由扁平化设计

之前玩飞冰的时候，建了一个vue的项目文件，打开看到router.js里写到路由扁平化，然后我就一直在查啥是路由扁平化，但是查来查去只查到网络扁平化，大致意思是简化网络，减少开销。  
然后……你们懂怎样路由扁平化设计了吗，我反正是有点懵懵的。  
后来我去查vue-router的一些路由设计，终于在一篇关于vue-router路由渲染的文章里大概明白了什么叫路由扁平化。  
其实也没多复杂，甚至还挺简单的。总结一下，就是把路由摊开来，是不是很符合扁平化这三个字（想到了摊鸡蛋饼）。说实际点，就是每个route都分开不设children。我觉得主要就是减少了渲染的开销吧，毕竟渲染子路由的同时还会把父路由也渲染了。  
比如：
```
const route = {
    path: '/list',
    name: 'list',
    component: List,
    children: [{
        path: 'datail/:id',
        name: 'detail',
        component: Detail
    }]
}
```
当访问二级路由detail的时候，List和Detail组件会同时渲染到页面，但是实际上我们并不需要List组件渲染进来。这时候最基础的做法就是路由扁平化，将list和detail拆成两个。（但是这里路由扁平化会破坏路由的父子语义，另外会导致面包屑导航无法使用，这里该怎么改在后面再说）  
路由扁平化大致就是如此吧，要是我说的有啥不对请dalao们一定告诉我！  

# beforeEach和afterEach的一些玩法

`beforeEach`是`vue-router`的全局前置守卫，接收三个参数：to，from，next。其中**一定要调用next方法来resolve钩子**。next可以改变导航。  
`afterEach`是`vue-router`的全局后置钩子，接收两个参数：to，from。与守卫不同的是钩子不会接收next也不会改变导航。  
以上是`beforeEach`和`afterEach`的基本知识，运用和`beforeEach`和`afterEach`能有一些有意思的玩法。  
比如：
## 长页面跳转自动回到顶部
有时候在浏览到一个很长的页面时，点击下一页就会自动跳转到页面顶部，这就可以用`afterEach`钩子函数实现。（要是啥都不写的话，到下一页还是停留在同样的位置）
实现如下：
```
router.afterEach((to, from) => {
    window.scrollTo(0, 0)
})
```
## 动态修改页面标题
标题是由`title`标签控制的，但是在一个SPA（单页面应用）中，切换到不同页面时，实际上`title`标签没有变（作为容器的html没变），因此要切换标题就需要用js来控制。  
我们可以在每个组件的`mounted`钩子函数中写个js来控制标题，但是页面增多就会增加维护成本。  
因此我们可以注册个全局守卫，在路由发生变化时，统一设置。（其实现在都是用`vue-meta`的吧，感觉会更方便点？）  
```
import Vue from 'vue'
import VueRouter from 'vue-router'

Vue.use(VueRouter)

const Routers = [{
    path: '/index',
    name: 'index',
    component: (resolve) => require(['./component/index/index.vue'], resolve),
    meta: {
        title: '首页'
    }
}]

const RouterConfig = {
    mode: 'history',
    routes: Routers
}

const router = new VueRouter(RouterConfig)

router.beforeEach((to, from, next) => {
    window.document.title = to.meta.title
    next()
})
```
在路由信息数组里设置一个`meta`数组，包含了`title`信息，然后再全局守卫里修改`title`为`meta`里的`title`信息，来实现动态修改标题。
## 登录验证
使用`localStorage`和`beforeEach`可以实现登录验证（但我觉得还是后端`jsonwebtoken`比较好？）。  
代码如下：
```
router.beforeEach((to, from, next) => {
    if(window.localStorage.getItem('token')) {
        next();
    } else {
        next('/login');
    }
})
```
`next()`入参，如果是false，会不导航；如果为路径，则会导航到指定路径下面。

## 响应路由变化
这两天写vue-ts的一个demo时，碰到一个很蛋疼的问题花了我好久时间，虽然后来发现是命名有问题，但我觉得还是可以记录一下这个问题的。  
vue-router文档上有写到，当同一个路由下切换子路由时，原来的组件实例会被复用。**因为两个路由都渲染同个组件，比起销毁再创建，复用则显得更加高效。不过，这也意味着组件的生命周期钩子不会再被调用。**  
这时候会发生，路由不响应的问题。要想动态响应，文档给出了一个方法：watch $route对象。这里记录下另一个方法，provide/inject方法。  
这个其实就是给子组件注入依赖，router-view绑定个`isRouterActive`状态的p判断，当路由切换时，调用methods里的`reload`方法实现刷新。代码如下： 
 
```
//父组件
export default{
    data() {
        return {
            isRouterActive: true
        }
    },
    provide() {
        return {
            reload: this.reload
        }
    },
    methods:{
        reload() {
            this.isRouterActive = false
            this.$nextTick(() => (this.isRouterActive = true))
        }
    }
}

//子路由
<template>
    <div class="main" @success="successHandle">
        <div class="info"><router-view/></div>
    </div>
</template>
<script>
export default{
    inject: ['reload'],
    methods: {
        async successHandle() {
            this.reload()
        }
    }
}
</script>
```
