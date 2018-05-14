﻿---
title: vue-router中使用EventBus传值需要注意到的问题
date: 2018-03-31T22:26:58.000Z
categories: 前端开发
tags:
  - vue-router
  - eventbus
---

 最近负责开发一个视频相关的项目，要用到vue-router，同时涉及到一些共有状态管理，但是少量的状态又不想用vuex，于是用到了EventBus，一般来说， 我们用EventBus的步骤如下：
<!-- more -->
- **首先新建一个js用来创建我们的EventBus，如Bus.js** 


```javascript
import Vue from 'vue';  
...
export default new Vue();
```

- **接着，我们在需要的地方通过$emit触发自定义事件，比如我这个时候有一个视频当前播放时间的状态需要传递**

```javascript
import Bus from '../components/Bus.js'
...
Bus.$emit('currentTime', 'time')
```

- **再然后就是我们在另一个路由页面(就分别叫页面A和B吧)通过$on监听自定义事件**

  ```javascript
  import Bus from '../components/Bus.js'
  ...
  Bus.$on('currentTime', (time) => console.log(time))
  ```

  通常情况下这个是应该可以打印出我们想要的数据，但是经我实验发现，当第一次通过路由跳转页面的时候控制台是没有任何输出的，只有第二次跳转开始控制台才有输出，然后我就去查了下资料，发现有这么一种说法:

  > vue-router切换的时候，会先加载新的组件，当新的组件渲染好但是还没mount的时候，销毁旧组件，然后再挂载新组件，也就是说当B页面的生命周期进行到beforeMount的时候，下一步走到的就是A页面的beforeDestory方法和接下去的destroyed方法

要知道我们一般都是在B页面的created方法里面去使用$on监听自定义事件，但是通过上面那段话我们知道，当我们在create方法里面监听事件的时候$emit事件已经发出去了，此时监听器还没有注册，那么要让$on监听到A页面的$emit发出的事件，可以在A页面的beforeDestory或destroyed去执行$emit，附上vue-router切换时候相关的生命周期顺序图： ![](https://www.tuchuang001.com/images/2018/04/27/5763769-1c04ab921c3d4876.png)

**[图片出自](https://www.jianshu.com/p/fde85549e3b0)**

但是我这个项目和这个人有不一样的地方，因为我需要留存上一个路由页面的状态，所以我在`router-view`的外面用了`keep-alive`属性，所以也就不存在A页面会走到beforeDestory和destoryed的说法，那么这时候该怎么做，我一开始是做了实验，既然我直接$emit那边接收不到，那我就延迟去$emit，一开始我用了个2s的延迟:

```javascript
import Bus from '../components/Bus.js'
...
setTimeout(() => {
   Bus.$emit('currentTime', 'time')
}, 2000)
```

目的就是让B页面$on执行开始监听事件的时候我再去$emit，发现是可以的，我又把时间减少到1s，依然正常运行，但是如果我们要去精确计算两个页面的切换时间岂不是太蠢，后来我就想到了vue自带的一个方法，敲黑板**vm.$nextTick( [callback] )**，这个方法的作用就是在dom更新之后异步执行回调方法，我在这个时候去$emit果然能够成功获取到数据，大功告成。

顺便说一句，如果路由外面没有使用keep-alive的话，你会发现随着切换次数增多$on监听事件执行的次数也越来越多，和你切换页面的次数成正比，尤大在issue里面说这是因为$on事件是不会自动清除的，也就是说你切换的次数越多$on监听也会越来越多，解决的方法是需要在B页面的beforeDestroy里面手动使用$off去关闭监听:

```javascript
import Bus from '../components/Bus.js'
beforeDestory() {
  Bus.$off('currentTime', (time) => console.log(time))
}
```

以上

参考链接
[vue2 eventbus 求解惑](https://segmentfault.com/q/1010000007879907) 
[vue中eventbus被多次触发（vue中使用eventbus踩过的坑）](https://www.jianshu.com/p/fde85549e3b0)
