# vue自定义指令

## 简介

在 Vue2.0 中，代码复用和抽象的主要形式是组件。然而，有的情况下，我们需要对普通 DOM 元素进行底层操作，就会用到自定义指令。

官方传送门：https://cn.vuejs.org/v2/guide/custom-directive.html

## 背景

例如：前端页面的按钮权限控制。就有以下几种方案

### 方案一

通过在标签添加自定义属性，如:code=“model:add“，然后使用vue的updated钩子监控dom的更新。获取带code属性的标签进行权限校验，代码如下：

只需添加code属性即可：

```
<Button type="primary" code="model:add">查询</Button>
```

main.vue:

```
    updated () {
      // 页面更新后校验按钮权限
      let tags = document.body.querySelectorAll('[code]')
      tags.forEach(tag => {
        // 如果没拥有改权限，屏蔽该按钮
        if (!this.user.authorities.includes(tag.getAttribute('code'))) {
          tag.style.display = 'none'
        }
      })
    },
```

**优点：**

1.可以渗透到html标签的任何一个角落

2.支持render函数

```
        renderHeader: (h, {column, index}) => {
          return h('Button', {
            attrs: {
              code: 'role:add'
            }
          })
        }
```

3.只需添加code属性即可，简略重复代码，专注业务开发

**缺点：**

1.使用了原生的js代码，对于vue而言不是很友好

2.vuex中的内容变更不会实时计算在页面展示(不过权限目前也只有重新登录才会刷新)

### 方案二

在需要进行权限控制的页面引入vuex，然后在需要的标签使用vue的原生指令v-if调用vuex的getters方法进行权限校验。代码如下：

```
// 引入vuex
import {mapGetters} from 'vuex'

// computed中指定要使用的计算函数
computed: {
  ...mapGetters('user', ['has'])
｝

// 标签通过v-if调用vuex的计算函数
<DropdownItem name="find" v-if="has('pushmsg:findOnePushMsgConfig')">查看</DropdownItem>
```

**优点：**

1.使用了vue的原生指令，无侵入

2.计算函数，vuex数据产生变化时实时改变生效

3.效率较方案一高

**缺点：**

1.重复代码，使用到权限的页面都需要引入一遍

2.每个页面添加权限，开发人员需要进行多不操作

### 方案三

通过前两种方案，我们就想：那么，有没有什么办法将方案一二的有点结合起来呢？答案就是：vue自定义指令

开发人员只需在代码中的任意标签使用定义好的全局指令v-has即可

```
<Button type="primary" @click="search" v-has="'moduleType:query'">查询</Button>
```

mian.js：

```
// 注册一个全局权限校验指令 `v-has`
Vue.directive('has', {
  update: function (el, binding, vnode) {
    if (!store.getters['user/has'](binding.value)) {
      el.parentNode.removeChild(el)
    }
  }
})
```

**优点：**

1.vue原生支持，无入侵

2.自定义提供多种触发方式，适应多种场景

3.代码简略，专注业务开发，减少重复引入重复代码

**缺点：**

1.不支持render函数



## 钩子函数

在阅读了官方文档后，可能会存在对钩子函数存在一些疑惑的地方。

- `bind`：只调用一次，指令第一次绑定到元素时调用。在这里可以进行一次性的初始化设置。
- `inserted`：被绑定元素插入父节点时调用 (仅保证父节点存在，但不一定已被插入文档中)。
- `update`：所在组件的 VNode 更新时调用，**但是可能发生在其子 VNode 更新之前**。指令的值可能发生了改变，也可能没有。但是你可以通过比较更新前后的值来忽略不必要的模板更新。
- `componentUpdated`：指令所在组件的 VNode **及其子 VNode** 全部更新后调用。
- `unbind`：只调用一次，指令与元素解绑时调用。

### bind与inserted区别

```
  <div v-has="a">
    <a v-has="b"></a>
  </div>
  
// 注册一个全局权限校验指令 `v-has`
Vue.directive('has', {
  update: function (el, binding, vnode) {
    if (!store.getters['user/has'](binding.value)) {
      el.parentNode.removeChild(el)
    }
  }
})
```

在vue的渲染过程中，步骤是：

	1.a标签绑定v-has指令
	
	2.把a标签插入到div中
	
	3.div的绑定可能与a标签绑定同步进行，如果div绑定完成了，插入所属的父标签或dom中。而a标签不管div是否绑定完成指令都会进行插入。

**bind钩子**：a标签此阶段进行指令绑定，但是还没插入到div中。通过指令el参数的父节点结果：

```
console.log(el.parentNode) //null
```

**inserted**：此步骤即第2步，此时获取a标签的parentNode结果：

```
console.log(el.parentNode) //<div>...</div>
```

【仅保证父节点存在，但不一定已被插入文档中】:官方这句话的意思就是第3步，通过a标签能获取到div元素，但是div元素还不一定绑定完成，所以可能还没插入到dom中。

### update与componentUpdated

#### 什么是vnode?

Vue.js 2.0中模拟DOM模型树就是VNode，Render函数执行后都会返回VNode对象。在上例中a标签绑定了v-has指令，vue在渲染之后，它产生一个vnode。

#### vnode的结构是怎样的？

通过console.log(this)打印出来的VueComponent我们就能看到vnode节点的信息。

![](https://github.com/yang-zhijiang/learn-data/blob/master/vuex/img/1-vnode%E7%BB%93%E6%9E%84.png?raw=true)

#### 触发原则

回过头来，update与componentUpdated触发条件就是vnode的属性发生变动或者是它的子vnode有更改。更新时触发update，所有节点更新完成后触发componentUpdated。

【组件的 VNode 更新时调用，**但是可能发生在其子 VNode 更新之前**】：现在看来这句话不难理解，以上例的div与a标签，意思是div和a的vnode同时存在数据更新，两个vnode的先后顺序问题而已。

#### 既然钩子是通过vnode更新触发，那像v-if的值在true与false之间切换如何触发呢？

在上图中，我们没有看到vnode存在任何指令的值，那值变化如何触发？我们再来看一张截图，一目了然

![](https://github.com/yang-zhijiang/learn-data/blob/master/vuex/img/1-vnode%E8%A7%A6%E5%8F%91.png?raw=true)



#### 方案三的触发原理

```
Vue.directive('has', {
  update: function (el, binding, vnode) {
    if (!store.getters['user/has'](binding.value)) {
      el.parentNode.removeChild(el)
    }
  }
})
```

指令中使用了vuex的计算函数getters，该函数效果与vue的computed相同。但是计算函数数据变更后实时计算触发的效果只在<template>标签范围内有效，不可能去触发我<script>标签中调用了该函数的js代码。那么，update钩子是如何被触发的呢？

层层挖掘，我们看到vnode里面包含了vuex的$store,而里面又包含了各种保存的数据。就是说vuex里保存的任一数据发生了更新，对于vue来说，vnode就是更新了，触发钩子。

![](https://github.com/yang-zhijiang/learn-data/blob/master/vuex/img/1-vnode%E6%9B%B4%E6%96%B01.png?raw=true)

![](https://github.com/yang-zhijiang/learn-data/blob/master/vuex/img/1-vnode%E6%9B%B4%E6%96%B02.png?raw=true)



### unbind

```
<a v-has="b" ref="a"></a>
```

unbind钩子我们该如何去触发？

```
this.$refs.a.$destroy() //其实就是调用该组件的销毁方法
```

组件销毁，解绑钩子触发。解绑后的a标签不再是vue组件，只是一个单纯的html元素而已。