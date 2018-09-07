# VUEX使用中的一些问题

## 什么是vuex？

Vuex 是一个专为 Vue.js 应用程序开发的**状态管理模式**。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。

请先阅读以下官方文档，传送门：https://vuex.vuejs.org/zh/

## 用途

简单来说，vuex的主要是用于不同组件间的通信，不同的主件只需通过this.$store即可获取vuex中保存的state数据。

**注意**：如果用户刷新页面，vuex中缓存的数据会清空。因为vuex的数据是保存在内存中的，会被释放。因此，它的主要作用并不是用于存储全局数据。

不过，对于我们后台系统而言，页面刷新是相对较少，只要合理利用，可以用来存储多个页面共享的数据，对于性能有一定的提升。

## 如何在js文件中使用？

```
import store from '@/store/index' //在js文件中引入即可

//获取state中数据的方式
store.getters //应通过getters来获取获取到的才是最新的。相当于vue的计算函数
store.state //如果通过state获取，数据只会是在引入时的数据而已，不是最新的
```

## 实战权限系统中的使用

### 背景

在权限系统中，目前主要分为四层：系统→分类→模块→权限。

分类配置页：读取系统列表

模块页：系统列表+分类列表

权限页：以上三个列表

这几层的数据几乎贯穿整个配置系统，每个页面几乎都反复去调用同一个接口获取列表数据，这是一个比较耗时的操作。

### 缓存共享数据

```
  //vuex中提供相关的方法
  state: {
    systems: '', // 缓存系统数据
    moduleTypeMap: new Map(), // 缓存功能分类数据
    moduleMap: new Map(), // 缓存模块数据
    permissionMap: new Map() // 缓存权限数据
  },
  mutations: {
    // 刷新功能分类缓存
    updateModuleTypeMap (state, type) {
      state.moduleTypeMap.set(type.id, type.data)
    },
    // 清空功能分类缓存
    clearModuleTypeMap (state, systemId) {
      state.moduleTypeMap.delete(systemId)
    }
  ｝,
  getters: {
    // 获取最新的系统数据
    getModuleTypeMap: state => {
      return state.moduleTypeMap
    }
  }
```

```
// 从缓存获取数据，获取不到则调用接口更新
module.cacheTypeList = function (systemId) {
  return new Promise(function (resolve) {
    // 查询缓存，存在数据，直接返回
    let types = store.getters.getModuleTypeMap
    if (types.has(systemId)) {
      resolve(JSON.parse(types.get(systemId)))
      return
    }

    // 找不到数据，则更新缓存再返回
    util.ajax.post('/module-type/list', {systemId: systemId}).then(res => {
      const data = res.data
      let types = []
      if (data.code === '200') {
        types = data.result.list
        if (types.length) {
          // 将数据缓存起来
          store.commit('updateModuleTypeMap', {id: systemId, data: JSON.stringify(types)})
        }
      }
      resolve(types)
    })
  })
}
```

