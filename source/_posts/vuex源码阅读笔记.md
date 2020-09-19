---
layout: post
title: vuex源码阅读笔记
date: 2020-09-14 15:36:42
tags: [vue, 笔记, vuex]
categories: vue
---

Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。

## 安装

```js
// vuex:
export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}
```

`vuex` 提供了一个 `install` 方法用来注册插件，我们 `use` 的时候就会执行，并把 `Vue` 作为参数传入，执行 `install` 会执行 `applyMixin(Vue)`，这个函数就是全局混入了 `beforeCreate` 钩子，在这个钩子中报错了 `this.$store = options.store`，并通过 `vm.parent` 把所有的组件都添加这个属性

## Store 实例化

```js
export default new Vuex.Store({
  actions,
  getters,
  state,
  mutations,
  modules
  // ...
})
```

实例化过程执行 Store 的 constructor，主要步骤有三步

1. `this._modules = new ModuleCollection(options)` 初始化模块
2. `installModule(this, state, [], this._modules.root)` 安装模块
3. `resetStoreVM(this, state)` 初始化 store.\_vm

### 初始化模块

vuex 设计成不仅仅可以在根实例上初始化状态，也允许分割成一个个 module，每个模块拥有自己的 state mutation action getter，而且可以无限嵌套

从数据结构上来看，模块的设计就是一个树形结构，store 本身可以理解为一个 root module，它下面的 modules 就是子模块， vux 需要完成这棵树的构建，构建的入口就是 `this._modules = new ModuleCollection(options)`

```js
export default class ModuleCollection {
  constructor(rawRootModule) {
    // register root module (Vuex.Store options)
    this.register([], rawRootModule, false)
  }

  get(path) {
    return path.reduce((module, key) => {
      return module.getChild(key)
    }, this.root)
  }

  getNamespace(path) {
    let module = this.root
    return path.reduce((namespace, key) => {
      module = module.getChild(key)
      return namespace + (module.namespaced ? key + '/' : '')
    }, '')
  }

  update(rawRootModule) {
    update([], this.root, rawRootModule)
  }

  register(path, rawModule, runtime = true) {
    if (process.env.NODE_ENV !== 'production') {
      assertRawModule(path, rawModule)
    }

    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      const parent = this.get(path.slice(0, -1))
      parent.addChild(path[path.length - 1], newModule)
    }

    // register nested modules
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }

  unregister(path) {
    const parent = this.get(path.slice(0, -1))
    const key = path[path.length - 1]
    if (!parent.getChild(key).runtime) return

    parent.removeChild(key)
  }
}
```

`ModuleCollection` 实例化的过程就是执行了 `register` 方法，它接受三个参数，`path` 表示路径， 表示定义模块的原始配置，`runtime` 表示是否是一个运行时创建的模块

`register` 方法首先通过 `const newModule = new Module(rawModule, runtime)` 创建了一个 `Module` 的实例，`Module` 是用来描述单个模块的类

```js
export default class Module {
  constructor(rawModule, runtime) {
    this.runtime = runtime
    // Store some children item
    this._children = Object.create(null)
    // Store the origin module object which passed by programmer
    this._rawModule = rawModule
    const rawState = rawModule.state

    // Store the origin module's state
    this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
  }

  get namespaced() {
    return !!this._rawModule.namespaced
  }

  addChild(key, module) {
    this._children[key] = module
  }

  removeChild(key) {
    delete this._children[key]
  }

  getChild(key) {
    return this._children[key]
  }

  update(rawModule) {
    this._rawModule.namespaced = rawModule.namespaced
    if (rawModule.actions) {
      this._rawModule.actions = rawModule.actions
    }
    if (rawModule.mutations) {
      this._rawModule.mutations = rawModule.mutations
    }
    if (rawModule.getters) {
      this._rawModule.getters = rawModule.getters
    }
  }

  forEachChild(fn) {
    forEachValue(this._children, fn)
  }

  forEachGetter(fn) {
    if (this._rawModule.getters) {
      forEachValue(this._rawModule.getters, fn)
    }
  }

  forEachAction(fn) {
    if (this._rawModule.actions) {
      forEachValue(this._rawModule.actions, fn)
    }
  }

  forEachMutation(fn) {
    if (this._rawModule.mutations) {
      forEachValue(this._rawModule.mutations, fn)
    }
  }
}
```

通过递归调用 register 函数，就会建议一颗完整的 module 树

### 安装模块

对模块中 state getters mutations actions 做初始化

```js
const state = this._modules.root.state
installModule(this, state, [], this._modules.root)
```

installModule 的定义

```js
function installModule(store, rootState, path, module, hot) {
  const isRoot = !path.length
  const namespace = store._modules.getNamespace(path)

  // register in namespace map
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }

  const local = (module.context = makeLocalContext(store, namespace, path))

  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```

`installModule` 支持五个参数，`store` 表示 `root` `store`；`state` 表示 `root` `state`，`path` 表示模块的访问路径；`module` 表示当前的模块，`hot` 表示是否热更新

默认情况下，模块内部的 `mutation` `action` `getter` 是注册在全局命名空间的，如果添加 `namespace:true`, 当模块被注册后，他的所有 `getter` `action` `mutation` 都会自动根据模块注册的路径调整命名

回到 `installModule` 方法，我们首先根据 `path` 获取 `namespace`，返回 / 拼接的字符串或者空字符串

```js
getNamespace (path) {
  let module = this.root
  return path.reduce((namespace, key) => {
    module = module.getChild(key)
    return namespace + (module.namespaced ? key + '/' : '')
  }, '')
}

pip install virtualenv  -i https://pypi.douban.com/simple
pip install apache-superset -i https://pypi.douban.com/simple
flask fab  create-admin
set FLASK_APP=superset
set FLASK_ENV=development
python superset db upgrade
python superset init
python superset runserver
```

接着 `namespaced` 为 `true` 则把 `module` 保存到 `_modulesNamespaceMap` `store._modulesNamespaceMap[namespace] = module`

接着判断 isroot 为 false 就把 parentState 和 module.state 设置属性关系 `Vue.set(parentState, moduleName, module.state)`，值得一提的是，`_withCommit` 保证是 `commit` 提交的，直接修改在 `strict` 模式下会报错

接着执行 `const local = module.context = makeLocalContext(store, namespace, path)` 生成 local 对象

```js
function makeLocalContext(store, namespace, path) {
  const noNamespace = namespace === ''

  const local = {
    dispatch: noNamespace
      ? store.dispatch
      : (_type, _payload, _options) => {
          const args = unifyObjectStyle(_type, _payload, _options)
          const { payload, options } = args
          let { type } = args

          if (!options || !options.root) {
            type = namespace + type
            if (process.env.NODE_ENV !== 'production' && !store._actions[type]) {
              console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
              return
            }
          }

          return store.dispatch(type, payload)
        },

    commit: noNamespace
      ? store.commit
      : (_type, _payload, _options) => {
          const args = unifyObjectStyle(_type, _payload, _options)
          const { payload, options } = args
          let { type } = args

          if (!options || !options.root) {
            type = namespace + type
            if (process.env.NODE_ENV !== 'production' && !store._mutations[type]) {
              console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
              return
            }
          }

          store.commit(type, payload, options)
        }
  }

  // getters and state object must be gotten lazily
  // because they will be changed by vm update
  Object.defineProperties(local, {
    getters: {
      get: noNamespace ? () => store.getters : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })

  return local
}
```

该方法定义了 `local` 对象，对于 `dispatch` 和 `commit` 方法，如果没有 `namespace`，它们就直接指向了 `root store` 的 `dispatch` 和 `commit` 方法，否则会创建方法，把 `type` 自动拼接上 `namespace`，然后执行 `store` 上对应的方法
对于 `getters` 而言，如果没有 `namespace`，则直接返回 `root store` 的 `getters`，否则返回 `makeLocalGetters(store, namespace)` 的返回值

```js
function makeLocalGetters(store, namespace) {
  const gettersProxy = {}
  const splitPos = namespace.length

  Object.keys(store.getters).forEach(type => {
    // skip if the target getter is not match this namespace
    if (type.slice(0, splitPos) !== namespace) return

    // extract local getter type
    const localType = type.slice(splitPos)

    // Add a port to the getters proxy.
    // Define as getter property because
    // we do not want to evaluate the getters in this time.
    Object.defineProperty(gettersProxy, localType, {
      get: () => store.getters[type],
      enumerable: true
    })
  })

  return gettersProxy
}
```

`makeLocalGetters` 首先获取 `namespace` 长度，然后遍历 `root store` 下所有的 `getters`, 如果符合 `namespace`, 就截取后面的 `localType`，用 `Object.defineProperty` 定义了 `gettersProxy`, 获取 `localType` 实际上是访问了 `store.getters[type]`

`state` 相对比较简单，就根据 `path` 查找 `state`

构造完 `local` 上下文后，我们回调 `installModule` 方法，接下来他会遍历模块中定义的`mutations actions getters` 分别执行他们的注册工作，
过程就是执行 `registerMutaions` 给 `rootStore` 的 `_mutations[types]` 添加 `wrappedMutationHandler` 方法，
执行 `registerActions` 给 `root store` 的 `_actions[types]` 添加 `wrappedActionHandler` 方法，
执行 `registerGetter` 给 `root store` 上的 `_wrappedGetters[key]` 指定 `wrappedGetter` 方法

再回到 `installModule` 方法，最后一步就是遍历模块中的所有子 `modules`，递归执行 `installModule` 方法

`installModule` 实际上就是完成了模块下的`state、getters、actions、mutations` 的初始化工作，并且通过递归遍历的方式，就完成了所有子模块的安装工作。

### 初始化 `store._vm`

```js
function resetStoreVM(store, state, hot) {
  const oldVm = store._vm

  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent

  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store)
  }

  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```

`resetStoreVM` 的作用就是想建立 `getters` 和 `state` 的联系，因为从设计上 `getters` 的获取就依赖了`state`，并且希望它的依赖能被缓存起来，且只有依赖值发生变化才会重新计算，这里直接用了 `vue` 的 `computed` 实现

`resetStoreVM` 首先遍历了 `_wrappedGetters` 获取每个 `getter` 的函数 `fn` 和 `key`，然后定义了 `computed[key] = () => fn(store)`, 这里的 `fn(store)` 就相当于执行 `wrapperGetter`, 返回的就是 `rawGetter` 执行函数，`rawGetter` 就是用户定义的 `getter` 函数，他的前两个参数是 `local state` 和 `local getters`，后两个参数就是 `root state` 和 `root getters`
接着实例化一个 `vue` 实例，`store._vm` 并把 `computed` 传入

```js
store._vm = new Vue({
  data: {
    $$state: state
  },
  computed
})
```

我们发现 data 选项里定义了 `$$state` 属性，而我们访问 `store.state` 的时候，实际上会访问 `store` 类上定义的 `state` 的 `get` 方法

```js
get state () {
  return this._vm._data.$$state
}
```

实际上就是访问的 `$$state`，那么 `getters` 和 `state` 是如何建立依赖逻辑的呢？

```js
forEachValue(wrappedGetters, (fn, key) => {
  // use computed to leverage its lazy-caching mechanism
  computed[key] = () => fn(store)
  Object.defineProperty(store.getters, key, {
    get: () => store._vm[key],
    enumerable: true // for local getters
  })
})
```

当我根据 `key` 访问 `store.getters` 的某一个 `getter` 的时候，实际上就是访问了 `store._vm[key]`，也就是 `computed[key]`，在执行 `computed[key]` 对应的函数的时候，会执行 `rawGetter(local.state,..)` 方法，那么就会访问到 `store.state`, 进而访问到 `$$state`, 这样就建立了一个依赖关系

当严格模式下，`store._vm` 会添加一个 `wathcer` 来观测 `this._data.$$state` 的变化，也就是当 `store.state` 被修改的时候, `store._committing` 必须为 `true`，否则在开发阶段会报警告。`store._committing` 默认值是 `false`，那么它什么时候会 `true` 呢，`Store` 定义了 `_withCommit` 实例方法：

```js
_withCommit (fn) {
  const committing = this._committing
  this._committing = true
  fn()
  this._committing = committing
}
```

它就是对 `fn` 包装了一个环境，确保在 `fn` 中执行任何逻辑的时候 `this._committing = true`。所以外部任何非通过 `Vuex` 提供的接口直接操作修改 `state` 的行为都会在开发阶段触发警告。

## API

对 `store` 做存取的操作

### 数据获取

``Vuex`最终存储的数据是在 state 上，我们之前分析过在`store.state`存储的是`root state`, 那么对于模块上的`state`，假设我们有 2个嵌套的`modules`，他们的`key`分别为`a`和`b`，我们可以 通过`store.state.a.b.xxx`的方式去获取`
它的实现发生在 `installModule` 的时候

```js
function installModule(store, rootState, path, module, hot) {
  const isRoot = !path.length

  // ...
  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }
  // ...
}
```

在递归执行 installModule 的过程中，递归执行了所有 getters 定义的注册，在之后的 resetStoreVM 过程中，执行了 store.getters 的初始化工作

```js
function installModule(store, rootState, path, module, hot) {
  // ...
  const namespace = store._modules.getNamespace(path)
  // ...
  const local = (module.context = makeLocalContext(store, namespace, path))

  // ...

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  // ...
}

function registerGetter(store, type, rawGetter, local) {
  if (store._wrappedGetters[type]) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(`[vuex] duplicate getter key: ${type}`)
    }
    return
  }
  store._wrappedGetters[type] = function wrappedGetter(store) {
    return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters // root getters
    )
  }
}

function resetStoreVM(store, state, hot) {
  // ...
  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  // ...
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  // ...
}
```

在 `installModule` 的过程中，建立了每个模块的上下文环境，因此当我们访问 `store.getters.xxx` 的时候，实际上就是执行 `rawGetter(local.state, local.getters, root.state, root.getters)`

### 数据存储

`Vuex` 对数据存储本质上就是对 `state` 做修改，并且只允许我们通过提交 `mutation` 的形式去修改 `state`，`mutation` 是一个函数

```js
mutations: {
  increment (state) {
    state.count++
  }
}
```

`mutations` 的初始化也是在 `installModule` 的时候

```js
function installModule(store, rootState, path, module, hot) {
  // ...
  const namespace = store._modules.getNamespace(path)

  // ...
  const local = (module.context = makeLocalContext(store, namespace, path))

  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })
  // ...
}

function registerMutation(store, type, handler, local) {
  const entry =
    store._mutations[type] ||
    (store._mutations[type] = []) -
      entry.push(function wrappedMutationHandler(payload) {
        handler.call(store, local.state, payload)
      })
}
```

`store` 提供了一个 `commit` 方法提交一个 `mutation`

```js
commit (_type, _payload, _options) {
  // check object-style commit
  const {
    type,
    payload,
    options
  } = unifyObjectStyle(_type, _payload, _options)

  const mutation = { type, payload }
  const entry = this._mutations[type]
  if (!entry) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(`[vuex] unknown mutation type: ${type}`)
    }
    return
  }
  this._withCommit(() => {
    entry.forEach(function commitIterator (handler) {
      handler(payload)
    })
  })
  this._subscribers.forEach(sub => sub(mutation, this.state))

  if (
    process.env.NODE_ENV !== 'production' &&
    options && options.silent
  ) {
    console.warn(
      `[vuex] mutation type: ${type}. Silent option has been removed. ` +
      'Use the filter functionality in the vue-devtools'
    )
  }
}
```

遍历 `this._mutations` 获取每个 `handler(payload)` 执行，实际就是执行了 `wrappedMutationHandler(payload)`，然后执行我们定义在 `mutation` 的函数, 传入 并 `local.state`

需要注意的是 `mutation` 必须是同步函数，因为我们有异步的需求，所以 `action` 也经常用到

`action` 类似于 `mutation`，不同在于 `action` 提交的是 `mutation`，而不是直接操作 `state`，并且它可以包含任意异步操作

```js
mutations: {
  increment (state) {
    state.count++
  }
},
actions: {
  increment (context) {
    setTimeout(() => {
      context.commit('increment')
    }, 0)
  }
}
```

actions 的初始化也是在 installModule 的时候

```js
function installModule(store, rootState, path, module, hot) {
  // ...
  const namespace = store._modules.getNamespace(path)

  // ...
  const local = (module.context = makeLocalContext(store, namespace, path))

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })
  // ...
}

function registerAction(store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler(payload, cb) {
    let res = handler.call(
      store,
      {
        dispatch: local.dispatch,
        commit: local.commit,
        getters: local.getters,
        state: local.state,
        rootGetters: store.getters,
        rootState: store.state
      },
      payload,
      cb
    )
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
}
```

`store` 提供了一个 `dispatch` 方法让我们提交一个 `action`

```js
dispatch (_type, _payload) {
  // check object-style dispatch
  const {
    type,
    payload
  } = unifyObjectStyle(_type, _payload)

  const action = { type, payload }
  const entry = this._actions[type]
  if (!entry) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(`[vuex] unknown action type: ${type}`)
    }
    return
  }

  this._actionSubscribers.forEach(sub => sub(action, this.state))

  return entry.length > 1
    ? Promise.all(entry.map(handler => handler(payload)))
    : entry[0](payload)
}
```

遍历 `store._actions` 中对应的数组，找个每个 `handler` 然后执行 `wrappedActionHandler(payload)`，接着会执行我们定义的 `action` 函数，
并传入了一个对象，包含当前模块的 `dispatch` `commit` `getters` `state` 以及全局的 `rootState` `rootGetters`, 所以我们定义的 `action` 函数能拿到
当前模块下的 `commit` 方法,

**因此 action 比我们自己写一个函数执行异步操作然后提交 mutation 的好处是在于它可以在参数中获取到当前模块的一些方法和状态，Vuex 帮我们做好了这些**

### 语法糖

#### mapState

用法：

```js
// 在单独构建的版本中辅助函数为 Vuex.mapState
import { mapState } from 'vuex'

export default {
  // ...
  computed: mapState({
    // 箭头函数可使代码更简练
    count: state => state.count,

    // 传字符串参数 'count' 等同于 `state => state.count`
    countAlias: 'count',

    // 为了能够使用 `this` 获取局部状态，必须使用常规函数
    countPlusLocalState(state) {
      return state.count + this.localCount
    }
  })
}
```

定义：

```js
export const mapState = normalizeNamespace((namespace, states) => {
  const res = {}
  normalizeMap(states).forEach(({ key, val }) => {
    res[key] = function mappedState() {
      let state = this.$store.state
      let getters = this.$store.getters
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapState', namespace)
        if (!module) {
          return
        }
        state = module.context.state
        getters = module.context.getters
      }
      return typeof val === 'function' ? val.call(this, state, getters) : state[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})

function normalizeNamespace(fn) {
  return (namespace, map) => {
    if (typeof namespace !== 'string') {
      map = namespace
      namespace = ''
    } else if (namespace.charAt(namespace.length - 1) !== '/') {
      namespace += '/'
    }
    return fn(namespace, map)
  }
}

function normalizeMap(map) {
  return Array.isArray(map) ? map.map(key => ({ key, val: key })) : Object.keys(map).map(key => ({ key, val: map[key] }))
}
```

当执行 `mapState(map)` 函数的时候，实际上就是执行 `normalizeNamespace` 包裹的函数，然后把 `map` 作为参数 `states` 传入，`mapState` 最终要构建一个对象，每个对象的元素都是一个方法，因为这个对象是要扩展到组件的 `computed` 属性的。
函数首先执行 `normalizeMap` 方法，把这个 `states` 变成一个数组，数组的每个元素都是 `key:value` 的形式，然后再遍历这个数组，以 `key` 作为对象的 `key`，值为一个 `mappedState` 函数，在这个 函数内部获取 `$store.state` 和 `$store.getters`,
如果 `val` 是函数就行，否则直接访问 `state[val]`

如果是模块中的 state，我们可以这样写

```js
computed: {
  mapState('some/nested/module', {
    a: state => state.a,
    b: state => state.b
  })
},
```

#### mapGetters

```js
import { mapGetters } from 'vuex'

export default {
  // ...
  computed: {
    // 使用对象展开运算符将 getter 混入 computed 对象中
    mapGetters([
      'doneTodosCount',
      'anotherGetter',
      // ...
    ])
  }
}
```

和 mapState 基本类似

```js
export const mapGetters = normalizeNamespace((namespace, getters) => {
  const res = {}
  normalizeMap(getters).forEach(({ key, val }) => {
    // thie namespace has been mutate by normalizeNamespace
    val = namespace + val
    res[key] = function mappedGetter() {
      if (namespace && !getModuleByNamespace(this.$store, 'mapGetters', namespace)) {
        return
      }
      if (process.env.NODE_ENV !== 'production' && !(val in this.$store.getters)) {
        console.error(`[vuex] unknown getter: ${val}`)
        return
      }
      return this.$store.getters[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```

#### mapMutaions

我们可以在组件中使用 `this.$store.commit('xx')` 提交 `mutation`，或者使用 `mapMutations` 辅助函数将组件中的 `methods` 映射为 `store.commit`

用法：

```js
import { mapMutations } from 'vuex'

export default {
  // ...
  methods: {
    ...mapMutations([
      'increment', // 将 `this.increment()` 映射为 `this.$store.commit('increment')`

      // `mapMutations` 也支持载荷：
      'incrementBy' // 将 `this.incrementBy(amount)` 映射为 `this.$store.commit('incrementBy', amount)`
    ]),
    ...mapMutations({
      add: 'increment' // 将 `this.add()` 映射为 `this.$store.commit('increment')`
    })
  }
}
```

mapMutations 支持传入一个数组或者对象，目标都是组件中对应的 methods 映射为 store.commit 的调用

定义：

```js
export const mapMutations = normalizeNamespace((namespace, mutations) => {
  const res = {}
  normalizeMap(mutations).forEach(({ key, val }) => {
    res[key] = function mappedMutation(...args) {
      // Get the commit method from store
      let commit = this.$store.commit
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapMutations', namespace)
        if (!module) {
          return
        }
        commit = module.context.commit
      }
      return typeof val === 'function' ? val.apply(this, [commit].concat(args)) : commit.apply(this.$store, [val].concat(args))
    }
  })
  return res
})
```

可以看到 `mappedMutation` 同样支持了 `namespace`, 并且支持了传入额外的参数 `args`，作为提交 `mutation` 的 `payload`, 最终就是执行了 `store.commit` 方法，并且这个 `commit` 会根据传入的 `namespace` 映射到对应 `module` 的 `commit` 上

#### mapActions

我们可以在组件中使用 `this.$store.dispatch('xx')` 提交 `action`，或者使用 `mapActions` 辅助函数将组件中的 `methods` 映射为 `store.dispatch` 的调用

定义，和 mapMutaions 基本一样

```js
export const mapActions = normalizeNamespace((namespace, actions) => {
  const res = {}
  normalizeMap(actions).forEach(({ key, val }) => {
    res[key] = function mappedAction(...args) {
      // get dispatch function from store
      let dispatch = this.$store.dispatch
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapActions', namespace)
        if (!module) {
          return
        }
        dispatch = module.context.dispatch
      }
      return typeof val === 'function' ? val.apply(this, [dispatch].concat(args)) : dispatch.apply(this.$store, [val].concat(args))
    }
  })
  return res
})
```

### 动态更新模块

在 `Vuex` 初始化阶段我们构造了模块树，初始化了模块上各个部分。在有一些场景下，我们需要动态去注入一些新的模块，`Vuex` 提供了模块动态注册功能，在 `store` 上提供了一个 `registerModule` 的 `API`。

```js
registerModule (path, rawModule, options = {}) {
  if (typeof path === 'string') path = [path]

  if (process.env.NODE_ENV !== 'production') {
    assert(Array.isArray(path), `module path must be a string or an Array.`)
    assert(path.length > 0, 'cannot register the root module by using registerModule.')
  }

  this._modules.register(path, rawModule)
  installModule(this, this.state, path, this._modules.get(path), options.preserveState)
  // reset store to update getters...
  resetStoreVM(this, this.state)
}
```

还有一个卸载的函数，当然这里只会卸载动态创建的模块

```js
unregisterModule (path) {
  if (typeof path === 'string') path = [path]

  if (process.env.NODE_ENV !== 'production') {
    assert(Array.isArray(path), `module path must be a string or an Array.`)
  }

  this._modules.unregister(path)
  this._withCommit(() => {
    const parentState = getNestedState(this.state, path.slice(0, -1))
    Vue.delete(parentState, path[path.length - 1])
  })
  resetStore(this)
}
```
