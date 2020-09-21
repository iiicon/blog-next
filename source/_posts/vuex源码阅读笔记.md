---
layout: post
title: vuex源码阅读笔记
date: 2020-09-14 15:36:42
tags: [vue2.x, 笔记, vuex]
categories: vue
---

## 安装

```js
// vuex:
export default {
  Store,
  install,
  version: "__VERSION__",
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers,
};
```

`vuex` 提供了一个 `install` 方法用来注册插件，我们 `use` 的时候就会执行，并把 `Vue` 作为参数传入，执行 `install` 会执行 `applyMixin(Vue)`，这个函数就是全局混入了 `beforeCreate` 钩子，在这个钩子中报错了 `this.$store = options.store`，并通过 `vm.parent` 把所有的组件都添加这个属性

## Store 实例化

```js
export default new Vuex.Store({
  actions,
  getters,
  state,
  mutations,
  modules,
  // ...
});
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
    this.register([], rawRootModule, false);
  }

  get(path) {
    return path.reduce((module, key) => {
      return module.getChild(key);
    }, this.root);
  }

  getNamespace(path) {
    let module = this.root;
    return path.reduce((namespace, key) => {
      module = module.getChild(key);
      return namespace + (module.namespaced ? key + "/" : "");
    }, "");
  }

  update(rawRootModule) {
    update([], this.root, rawRootModule);
  }

  register(path, rawModule, runtime = true) {
    if (process.env.NODE_ENV !== "production") {
      assertRawModule(path, rawModule);
    }

    const newModule = new Module(rawModule, runtime);
    if (path.length === 0) {
      this.root = newModule;
    } else {
      const parent = this.get(path.slice(0, -1));
      parent.addChild(path[path.length - 1], newModule);
    }

    // register nested modules
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime);
      });
    }
  }

  unregister(path) {
    const parent = this.get(path.slice(0, -1));
    const key = path[path.length - 1];
    if (!parent.getChild(key).runtime) return;

    parent.removeChild(key);
  }
}
```

`ModuleCollection` 实例化的过程就是执行了 `register` 方法，它接受三个参数，`path` 表示路径， 表示定义模块的原始配置，`runtime` 表示是否是一个运行时创建的模块

`register` 方法首先通过 `const newModule = new Module(rawModule, runtime)` 创建了一个 `Module` 的实例，`Module` 是用来描述单个模块的类

```js
export default class Module {
  constructor(rawModule, runtime) {
    this.runtime = runtime;
    // Store some children item
    this._children = Object.create(null);
    // Store the origin module object which passed by programmer
    this._rawModule = rawModule;
    const rawState = rawModule.state;

    // Store the origin module's state
    this.state = (typeof rawState === "function" ? rawState() : rawState) || {};
  }

  get namespaced() {
    return !!this._rawModule.namespaced;
  }

  addChild(key, module) {
    this._children[key] = module;
  }

  removeChild(key) {
    delete this._children[key];
  }

  getChild(key) {
    return this._children[key];
  }

  update(rawModule) {
    this._rawModule.namespaced = rawModule.namespaced;
    if (rawModule.actions) {
      this._rawModule.actions = rawModule.actions;
    }
    if (rawModule.mutations) {
      this._rawModule.mutations = rawModule.mutations;
    }
    if (rawModule.getters) {
      this._rawModule.getters = rawModule.getters;
    }
  }

  forEachChild(fn) {
    forEachValue(this._children, fn);
  }

  forEachGetter(fn) {
    if (this._rawModule.getters) {
      forEachValue(this._rawModule.getters, fn);
    }
  }

  forEachAction(fn) {
    if (this._rawModule.actions) {
      forEachValue(this._rawModule.actions, fn);
    }
  }

  forEachMutation(fn) {
    if (this._rawModule.mutations) {
      forEachValue(this._rawModule.mutations, fn);
    }
  }
}
```

通过递归调用 register 函数，就会建议一颗完整的 module 树

### 安装模块

对模块中 state getters mutations actions 做初始化

```js
const state = this._modules.root.state;
installModule(this, state, [], this._modules.root);
```

installModule 的定义

```js
function installModule(store, rootState, path, module, hot) {
  const isRoot = !path.length;
  const namespace = store._modules.getNamespace(path);

  // register in namespace map
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module;
  }

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1));
    const moduleName = path[path.length - 1];
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state);
    });
  }

  const local = (module.context = makeLocalContext(store, namespace, path));

  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key;
    registerMutation(store, namespacedType, mutation, local);
  });

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key;
    const handler = action.handler || action;
    registerAction(store, type, handler, local);
  });

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key;
    registerGetter(store, namespacedType, getter, local);
  });

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot);
  });
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
  const noNamespace = namespace === "";

  const local = {
    dispatch: noNamespace
      ? store.dispatch
      : (_type, _payload, _options) => {
          const args = unifyObjectStyle(_type, _payload, _options);
          const { payload, options } = args;
          let { type } = args;

          if (!options || !options.root) {
            type = namespace + type;
            if (
              process.env.NODE_ENV !== "production" &&
              !store._actions[type]
            ) {
              console.error(
                `[vuex] unknown local action type: ${args.type}, global type: ${type}`
              );
              return;
            }
          }

          return store.dispatch(type, payload);
        },

    commit: noNamespace
      ? store.commit
      : (_type, _payload, _options) => {
          const args = unifyObjectStyle(_type, _payload, _options);
          const { payload, options } = args;
          let { type } = args;

          if (!options || !options.root) {
            type = namespace + type;
            if (
              process.env.NODE_ENV !== "production" &&
              !store._mutations[type]
            ) {
              console.error(
                `[vuex] unknown local mutation type: ${args.type}, global type: ${type}`
              );
              return;
            }
          }

          store.commit(type, payload, options);
        },
  };

  // getters and state object must be gotten lazily
  // because they will be changed by vm update
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace),
    },
    state: {
      get: () => getNestedState(store.state, path),
    },
  });

  return local;
}
```

该方法定义了 `local` 对象，对于 `dispatch` 和 `commit` 方法，如果没有 `namespace`，它们就直接指向了 `root store` 的 `dispatch` 和 `commit` 方法，否则会创建方法，把 `type` 自动拼接上 `namespace`，然后执行 `store` 上对应的方法
对于 `getters` 而言，如果没有 `namespace`，则直接返回 `root store` 的 `getters`，否则返回 `makeLocalGetters(store, namespace)` 的返回值

```js
function makeLocalGetters(store, namespace) {
  const gettersProxy = {};

  const splitPos = namespace.length;
  Object.keys(store.getters).forEach((type) => {
    // skip if the target getter is not match this namespace
    if (type.slice(0, splitPos) !== namespace) return;

    // extract local getter type
    const localType = type.slice(splitPos);

    // Add a port to the getters proxy.
    // Define as getter property because
    // we do not want to evaluate the getters in this time.
    Object.defineProperty(gettersProxy, localType, {
      get: () => store.getters[type],
      enumerable: true,
    });
  });

  return gettersProxy;
}
```

`makeLocalGetters` 首先获取 `namespace` 长度，然后遍历 `root store` 下所有的 `getters`, 如果符合 `namespace`, 就截取后面的 `localType`，用 `Object.defineProperty` 定义了 `gettersProxy`, 获取 `localType` 实际上是访问了 `store.getters[type]`

`state` 相对比较简单，就根据 `path` 查找 `state`

构造完 `local` 上下文后，我们回调 `installModule` 方法，接下来他会遍历模块中定义的` mutations actions getters` 分别执行他们的注册工作，
过程就是执行 `registerMutaions` 给 `rootStore` 的 `_mutations[types]` 添加 `wrappedMutationHandler` 方法，
执行 `registerActions` 给 `root store` 的 `_actions[types]` 添加 `wrappedActionHandler ` 方法，
执行 `registerGetter` 给 `root store` 上的 `_wrappedGetters[key]` 指定 `wrappedGetter ` 方法

再回到 `installModule` 方法，最后一步就是遍历模块中的所有子 `modules`，递归执行 `installModule` 方法

`installModule` 实际上就是完成了模块下的` state、getters、actions、mutations` 的初始化工作，并且通过递归遍历的方式，就完成了所有子模块的安装工作。

### 初始化 `store._vm`

```js
function resetStoreVM(store, state, hot) {
  const oldVm = store._vm;

  // bind store public getters
  store.getters = {};
  const wrappedGetters = store._wrappedGetters;
  const computed = {};
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store);
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true, // for local getters
    });
  });

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent;
  Vue.config.silent = true;
  store._vm = new Vue({
    data: {
      $$state: state,
    },
    computed,
  });
  Vue.config.silent = silent;

  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store);
  }

  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null;
      });
    }
    Vue.nextTick(() => oldVm.$destroy());
  }
}
```

`resetStoreVM` 的作用就是想建立 `getters` 和 `state` 的联系，因为从设计上 `getters` 的获取就依赖了` state`，并且希望它的依赖能被缓存起来，且只有依赖值发生变化才会重新计算，这里直接用了 `vue` 的 `computed` 实现

`resetStoreVM` 首先遍历了 `_wrappedGetters` 获取每个 `getter` 的函数 `fn` 和 `key`，然后定义了 `computed[key] = () => fn(store)`, 这里的 `fn(store)` 就相当于执行 `wrapperGetter`, 返回的就是 `rawGetter` 执行函数，`rawGetter` 就是用户定义的 `getter` 函数，他的前两个参数是 `local state` 和 `local getters`，后两个参数就是 `root state` 和 `root getters`
接着实例化一个 `vue` 实例，`store._vm` 并把 `computed` 传入

```js
store._vm = new Vue({
  data: {
    $$state: state,
  },
  computed,
});
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
  computed[key] = () => fn(store);
  Object.defineProperty(store.getters, key, {
    get: () => store._vm[key],
    enumerable: true, // for local getters
  });
});
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

### 数据获取

### 数据存储

### 语法糖

### 动态更新模块0