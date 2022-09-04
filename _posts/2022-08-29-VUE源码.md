---
layout: post
title:  "Vue源码"
date:   2022-08-29 19:28:51 +0800
categories: Vue
---

## 一、rollup配置

> vue 库采用 rollup 进行打包
>
> 1. npm init
> 2. 安装依赖
> 3. 配置 rollup.config.js 文件
> 4. 配置 babel
> 5. 添加运行脚本
> 6. 创建 src/index.js
> 7. 创建 public/index.html 

## 二、源码分析

###  1. vue-options

+ 把 data 等数据通过选项对象传递给 Vue 构造函数，构造函数接收参数并传递给`initMixin()`方法,内部通过 `_init()` 接收。
+  `_init()` 中真正的是通过 `initState()` 中的 `initData（）`进行初始化 data 。
+ 判断 data 数据类型，如果是函数，那么就通过 `data.call（vm）`调用，如果是对象，直接返回对象，接下来将要对数据进行劫持。

### 1. 数据劫持

+ 在 `initData()` 中调用 `observer()` 把数据传递过去。
+ 先判断数据是否为对象，不为对象不进行劫持，若为对象则传递到`new Oberver()` 中的`walk()` 中进行对象循环。
+ 再调用`defineReactive()` 对每一个数据劫持，继续调用`obverser()` 对新增的数据进行劫持。
+ 处理直接通过 this 访问到 data 中的数据，在`initData()` 中调用 `proxy()` 对vm进行劫持，当访问vm的数据时，将 vm 上的data代理到 _data 上，再次触发 `observer()` 进行数据劫持。
+ 在`Observer()` 中给需要劫持的数据添加一个 **-ob-** ,标识是否被劫持。 
+ ==数组劫持== 
  + 在`Observer()` 对数组进行判断，如果是数组，则调用`observeArray()` 方法，对数组中的复杂数据类型进行劫持(触发`observer()`) 
  + 重写 7 个数组方法，保留自身功能，对 push、unshift、splice 方法增加的数据进行劫持，最后调用`notify()` 进行通知更新。


![初始化和劫持](https://cdn.jsdelivr.net/gh/TCIano/blog_img/importcdn.png)

### 2. 模板编译

+ 当页面初始化完成并且当前数据都变为响应式时，开始编译模板。

+ 在`compileToFunction` 中通过`parseHtml` 方法解析模板，通过正则匹配来编译模板，生成AST语法树。
+ 将AST语法树转化为`render` 函数，生成虚拟 DOM ，虚拟 DOM进行对比，根据`patch` 方法进行 DOM 的更新。

### 3. 数据响应式(模板更新)

+ 创建了一个 Watcher，内部通过 get 调用`updateComponent()` 重新更新编译整个组件，内部进行新旧DOM树的对比。
+ 收集 Watcher ：编译模板中用到哪些数据，就给哪些数据收集 Watcher ，通过 dep 进行 Watcher 收集，每一个数据通过闭包的方式都有自己的 dep ，通过  dep 进行 Watcher 收集，所以每个数据都有了自己的 Watcher ，在数据劫持中的 get 中收集 Watcher ，拿到 Watcher ，通过 `Dep.target ` 进行中转，Watcher 中的 getter 调用前将this 传递给Dep.target 然后在 get 中 将`Dep.target ` 进行收集。
+ 通知更新：数据发生变化时会触发`set()` ,然后获取到数据对应的`dep` ，通过 subs 存储收集到的 wathcer , `dep` 中通过`notify()` 循环收集到的watcher，然后调用watcher的`update()` 方法进行组件更新，其实是调用了 watcher 中的 `get()` 然后通过内部的`getter()` 里的`updateComponent` 进行组件更新。
+ 其实Vue中的数据响应式为多对多的模式，通过 dep 收集 watcher 因为每个数据可能对应多个 watcher， 也要通过 watcher 收集 dep 因为每个组件中有多个数据。在`dep` 中调用`depend` ，内部是通过`addDep()` 来收集`Dep`，通过`addSubs()` 收集watcher

![模板更新](https://cdn.jsdelivr.net/gh/TCIano/blog_img/%E6%BA%90%E7%A0%81%E6%95%B0%E6%8D%AE%E5%93%8D%E5%BA%94%E5%BC%8F.png)

## 4. 异步更新视图

让所有的数据更新完成，再更新视图。

+ 当数据更新时，调用`quereWatcher()`通过 `quereWatcher()` 对 watcher 队列进行收集，根据watcher上的id属性来判断当前watcher是否被标记，把未被标记的watcher 加入到异步队列中，再调用 watcher 中的 `get()`方法，如果同步调用，那么只会更新第一个数据的视图，因此需要所有数据更新完统一更新视图，所以要把更新操作放到${\textcolor{red}{异步任务}}$中。
+ 把更新操作放到`nextTick()` 中执行，其实`this.$nextTick()` 就是异步更新视图中的`nextTick()` ,优先使用微任务，因为同步任务完成之后进入微任务更快，当不支持 Promise 的时候，使用`mutationsobserver()` (也是微任务)，如果不支持`mutationsobserver()` 那么就用`setImmediate()` （宏任务,比setTimeout快），如果也不支持`setImmediate()` 那么就用`setTimeout()` 
