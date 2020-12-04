---
title: vue3之CompositionAPI相关
date: 2020-08-20 10:03:35
tags: [vue3, 笔记]
categories: vue
---

# 组件渲染前的初始化过程

Vue.js 3.0 允许我们在编写组件的时候添加一个 setup 启动函数，它是 Composition API 逻辑组织的入口

```js
<template>
  <button @click="increment">
    Count is: {{ state.count }}, double is: {{ state.double }}
  </button>
</template>
<script>
import { reactive, computed } from 'vue'
export default {
  setup() {
    const state = reactive({
      count: 0,
      double: computed(() => state.count * 2)
    })
    function increment() {
      state.count++
    }
    return {
      state,
      increment
    }
  }
}

</script>
```

**这里的 state 和 increment 包含在 setup 函数的返回对象中，那么他是如何模板中引用的？**

在 `vue.js2.x` 编写组件的时候，会在 `props、data、methods、computed` 等 `options` 中定义一些变量。在组件初始化阶段，`vue.js` 内部会处理这些 `options`，即把定义的变量添加到了组件实例上。等模板编译成 `render` 函数的时候，内部通过 `with(this){}` 的语法去访问在组件实例中的变量

`vue3` 的 `setup` 函数的执行逻辑是在设置组件实例的时候处理的

## 创建和设置组件实例

组件的渲染流程是 创建 `vnode` 渲染 `vnode` 和生成 `dom`

其中渲染 vnode 的过程主要就是在挂载组件

```js
const mountComponent = (
  initialVNode,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized
) => {
  // 创建组件实例
  const instance = (initialVNode.component = createComponentInstance(
    initialVNode,
    parentComponent,
    parentSuspense
  ));
  // 设置组件实例
  setupComponent(instance);
  // 设置并运行带副作用的渲染函数
  setupRenderEffect(
    instance,
    initialVNode,
    container,
    anchor,
    parentSuspense,
    isSVG,
    optimized
  );
};
```

先看创建组件实例的过程，这个过程主要是执行了 `createComponentInstance` 创建组件实例：

```js
function createComponentInstance(vnode, parent, suspense) {
  // 继承父组件实例上的 appContext，如果是根组件，则直接从根 vnode 中取。

  const appContext =
    (parent ? parent.appContext : vnode.appContext) || emptyAppContext;

  const instance = {
    // 组件唯一 id
    uid: uid++,
    // 组件 vnode
    vnode,
    // 父组件实例
    parent,
    // app 上下文
    appContext,
    // vnode 节点类型
    type: vnode.type,
    // 根组件实例
    root: null,
    // 新的组件 vnode
    next: null,
    // 子节点 vnode
    subTree: null,
    // 带副作用更新函数
    update: null,
    // 渲染函数
    render: null,
    // 渲染上下文代理
    proxy: null,
    // 带有 with 区块的渲染上下文代理
    withProxy: null,
    // 响应式相关对象
    effects: null,
    // 依赖注入相关
    provides: parent ? parent.provides : Object.create(appContext.provides),
    // 渲染代理的属性访问缓存
    accessCache: null,
    // 渲染缓存
    renderCache: [],
    // 渲染上下文
    ctx: EMPTY_OBJ,
    // data 数据
    data: EMPTY_OBJ,
    // props 数据
    props: EMPTY_OBJ,
    // 普通属性
    attrs: EMPTY_OBJ,
    // 插槽相关
    slots: EMPTY_OBJ,
    // 组件或者 DOM 的 ref 引用
    refs: EMPTY_OBJ,
    // setup 函数返回的响应式结果
    setupState: EMPTY_OBJ,
    // setup 函数上下文数据
    setupContext: null,
    // 注册的组件
    components: Object.create(appContext.components),
    // 注册的指令
    directives: Object.create(appContext.directives),
    // suspense 相关
    suspense,
    // suspense 异步依赖
    asyncDep: null,
    // suspense 异步依赖是否都已处理
    asyncResolved: false,
    // 是否挂载
    isMounted: false,
    // 是否卸载
    isUnmounted: false,
    // 是否激活
    isDeactivated: false,
    // 生命周期，before create
    bc: null,
    // 生命周期，created
    c: null,
    // 生命周期，before mount
    bm: null,
    // 生命周期，mounted
    m: null,
    // 生命周期，before update
    bu: null,
    // 生命周期，updated
    u: null,
    // 生命周期，unmounted
    um: null,
    // 生命周期，before unmount
    bum: null,
    // 生命周期, deactivated
    da: null,
    // 生命周期 activated
    a: null,
    // 生命周期 render triggered
    rtg: null,
    // 生命周期 render tracked
    rtc: null,
    // 生命周期 error captured
    ec: null,
    // 派发事件方法
    emit: null,
  };

  // 初始化渲染上下文
  instance.ctx = { _: instance };
  // 初始化根组件指针
  instance.root = parent ? parent.root : instance;
  // 初始化派发事件方法
  instance.emit = emit.bind(null, instance);
  return instance;
}
```

`vue2` 通过 `new Vue`来初始化一个组件实例，到了 `vue3` 我们直接通过创建对象去创建组件的实例，这两种方法并无本质的区别，都是引用一个对象，在整个组件的生命周期中去维护组件的状态数据和上下文环境

创建好 `instance` 实例后，接下来就是设置它的一些属性，目前已完成了组件的上下文，根组件指针以及派发事件方法的设置

接着是组件实例的设置流程，对 `setup` 函数的处理就在这里完成，主要就是执行 `setComponent`：

```js
function setupComponent(instance, isSSR = false) {
  const { props, children, shapeFlag } = instance.vnode;
  // 判断是否是一个有状态的组件
  const isStateful = shapeFlag & 4;
  // 初始化 props
  initProps(instance, props, isStateful, isSSR);
  // 初始化 插槽
  initSlots(instance, children);
  // 设置有状态的组件实例
  const setupResult = isStateful
    ? setupStatefulComponent(instance, isSSR)
    : undefined;

  return setupResult;
}
```

这里从组件 `vnode` 中获取了 `props` `children` `shapeFlag` 等属性，然后分别对 `props` 和插槽进行初始化，然后根据 `shapeFlag` 的值，我们判断如果是一个有状态的组件，就要去设置
执行 `setupStatefulComponent` 函数，它主要做了三件事，创建渲染上下文代理、判断处理 setup 函数、完成组件实例设置

```js
function setupStatefulComponent(instance, isSSR) {
  const Component = instance.type;
  // 创建渲染代理的属性访问缓存
  instance.accessCache = {};
  // 创建渲染上下文代理
  instance.proxy = new Proxy(instance.ctx, PublicInstanceProxyHandlers);
  // 判断处理 setup 函数
  const { setup } = Component;
  if (setup) {
    // 如果 setup 函数带参数，则创建一个 setupContext
    const setupContext = (instance.setupContext =
      setup.length > 1 ? createSetupContext(instance) : null);
    // 执行 setup 函数，获取结果
    const setupResult = callWithErrorHandling(
      setup,
      instance,
      0 /* SETUP_FUNCTION */,
      [instance.props, setupContext]
    );
    // 处理 setup 执行结果
    handleSetupResult(instance, setupResult);
  } else {
    // 完成组件实例设置
    finishComponentSetup(instance);
  }
}
```

### 创建渲染上下文代理

这一步就是对 `instance.ctx` 做了代理

`vue2` 在初始化组件的时候，`data` 中定义的 `msg` 在组件内部是存储在 `this._data` 上的，而模板渲染的时候访问 `this.msg` 实际上是访问 `this._data.msg`，这是因为访问 `data` 的时候做了一层代理

`vue3` 为了方便维护，我们把组件中不同状态的数据存储到不同的属性中，比如存储到 `setupState` `ctx` `data` `props` 中。我们在执行组件渲染函数的时候，为了方便用户使用，会直接访问渲染上下文 `instance.ctx`，所以这里的代理就是对渲染上下文 `instance.ctx` 属性的访问和修改，代理到对 `setupState` `ctx` `data` `props` 中的数据的访问和修改

接下了分析一下 proxy 的几个方法 get has set

#### get

```js
const PublicInstanceProxyHandlers = {
  get({ _: instance }, key) {
    const {
      ctx,
      setupState,
      data,
      props,
      accessCache,
      type,
      appContext,
    } = instance;
    if (key[0] !== "$") {
      // setupState / data / props / ctx
      // 渲染代理的属性访问缓存中
      const n = accessCache[key];
      if (n !== undefined) {
        // 从缓存中取
        switch (n) {
          case 0 /* SETUP */:
            return setupState[key];
          case 1 /* DATA */:
            return data[key];
          case 3 /* CONTEXT */:
            return ctx[key];
          case 2 /* PROPS */:
            return props[key];
        }
      } else if (setupState !== EMPTY_OBJ && hasOwn(setupState, key)) {
        accessCache[key] = 0;
        // 从 setupState 中取数据
        return setupState[key];
      } else if (data !== EMPTY_OBJ && hasOwn(data, key)) {
        accessCache[key] = 1;
        // 从 data 中取数据
        return data[key];
      } else if (
        type.props &&
        hasOwn(normalizePropsOptions(type.props)[0], key)
      ) {
        accessCache[key] = 2;
        // 从 props 中取数据
        return props[key];
      } else if (ctx !== EMPTY_OBJ && hasOwn(ctx, key)) {
        accessCache[key] = 3;
        // 从 ctx 中取数据
        return ctx[key];
      } else {
        // 都取不到
        accessCache[key] = 4;
      }
    }
    const publicGetter = publicPropertiesMap[key];
    let cssModule, globalProperties;
    // 公开的 $xxx 属性或方法
    if (publicGetter) {
      return publicGetter(instance);
    } else if (
      // css 模块，通过 vue-loader 编译的时候注入
      (cssModule = type.__cssModules) &&
      (cssModule = cssModule[key])
    ) {
      return cssModule;
    } else if (ctx !== EMPTY_OBJ && hasOwn(ctx, key)) {
      // 用户自定义的属性，也用 `$` 开头
      accessCache[key] = 3;
      return ctx[key];
    } else if (
      // 全局定义的属性
      ((globalProperties = appContext.config.globalProperties),
      hasOwn(globalProperties, key))
    ) {
      return globalProperties[key];
    } else if (
      process.env.NODE_ENV !== "production" &&
      currentRenderingInstance &&
      key.indexOf("__v") !== 0
    ) {
      if (data !== EMPTY_OBJ && key[0] === "$" && hasOwn(data, key)) {
        // 如果在 data 中定义的数据以 $ 开头，会报警告，因为 $ 是保留字符，不会做代理
        warn(
          `Property ${JSON.stringify(
            key
          )} must be accessed via $data because it starts with a reserved ` +
            `character and is not proxied on the render context.`
        );
      } else {
        // 在模板中使用的变量如果没有定义，报警告
        warn(
          `Property ${JSON.stringify(key)} was accessed during render ` +
            `but is not defined on instance.`
        );
      }
    }
  },
};
```

可以看到，函数首先判断 `key` 不以 `$` 开头的情况，这部分数据可能是 `setupState` `data` `props` `ctx` 中的一种，其中 `data` `props` 我们已经很熟悉了，`setupState` 就是 `setup` 函数返回的数据，`ctx` 包括了计算属性，组件方法和用户自定义的一些数据

如果 `key` 不以 `$` 开头，那么就依次判断 `setupState data props ctx` 中是否包含这个 `key`，如果包含就返回对应值

#### set

```js
const PublicInstanceProxyHandlers = {
  set({ _: instance }, key, value) {
    const { data, setupState, ctx } = instance;
    if (setupState !== EMPTY_OBJ && hasOwn(setupState, key)) {
      // 给 setupState 赋值
      setupState[key] = value;
    } else if (data !== EMPTY_OBJ && hasOwn(data, key)) {
      // 给 data 赋值
      data[key] = value;
    } else if (key in instance.props) {
      // 不能直接给 props 赋值
      process.env.NODE_ENV !== "production" &&
        warn(
          `Attempting to mutate prop "${key}". Props are readonly.`,
          instance
        );
      return false;
    }

    if (key[0] === "$" && key.slice(1) in instance) {
      // 不能给 Vue 内部以 $ 开头的保留属性赋值
      process.env.NODE_ENV !== "production" &&
        warn(
          `Attempting to mutate public property "${key}". ` +
            `Properties starting with $ are reserved and readonly.`,
          instance
        );
      return false;
    } else {
      // 用户自定义数据赋值
      ctx[key] = value;
    }
    return true;
  },
};
```

`set` 函数主要做的事情就是对渲染上下文 `instance.ctx` 中的属性赋值，它实际上是代理到对应的数据类型中去完成赋值操作的。这里仍然要注意顺序问题，和 `get` 一样，优先判断 `setupState`，然后是 `data`，接着是 `props`

#### has

```js
const PublicInstanceProxyHandlers = {
  has({ _: { data, setupState, accessCache, ctx, type, appContext } }, key) {
    // 依次判断
    return (
      accessCache[key] !== undefined ||
      (data !== EMPTY_OBJ && hasOwn(data, key)) ||
      (setupState !== EMPTY_OBJ && hasOwn(setupState, key)) ||
      (type.props && hasOwn(normalizePropsOptions(type.props)[0], key)) ||
      hasOwn(ctx, key) ||
      hasOwn(publicPropertiesMap, key) ||
      hasOwn(appContext.config.globalProperties, key)
    );
  },
};
```

这个函数的实现很简单，依次判断 `key` 是否存在于 `accessCache、data、setupState、props` 、用户数据、公开属性以及全局属性中，然后返回结果。

### 判断处理 setup 函数

```js
// 判断处理 setup 函数
const { setup } = Component;
if (setup) {
  // 如果 setup 函数带参数，则创建一个 setupContext
  const setupContext = (instance.setupContext =
    setup.length > 1 ? createSetupContext(instance) : null);
  // 执行 setup 函数获取结果
  const setupResult = callWithErrorHandling(
    setup,
    instance,
    0 /* SETUP_FUNCTION */,
    [instance.props, setupContext]
  );
  // 处理 setup 执行结果
  handleSetupResult(instance, setupResult);
}
```

如果组件中定义了 `setup` 函数，接下来就是处理 `setup` 函数的流程，主要是三个步骤：创建 `setup` 函数上下文，执行 `setup` 函数并获取结果和处理 `setup` 函数的执行结果

首先判断 setup 函数的参数长度，如果大于 1，就创建 setupContext 上下文

```js
const setupContext = (instance.setupContext =
  setup.length > 1 ? createSetupContext(instance) : null);

function createSetupContext(instance) {
  return {
    attrs: instance.attrs,
    slots: instance.slots,
    emit: instance.emit,
  };
}
```

使用如下

```js
// 子组件
<template>
  <p>{{ msg }}</p>
  <button @click="onClick">Toggle</button>
</template>
<script>
  export default {
    props: {
      msg: String
    },
    setup (props, { emit }) {
      function onClick () {
        emit('toggle')
      }
      return {
        onClick
      }
    }
  }
</script>

// 父组件
<template>
  <HelloWorld @toggle="toggle" :msg="msg"></HelloWorld>
</template>
<script>
  import { ref } from 'vue'
  import HelloWorld from "./components/HelloWorld";
  export default {
    components: { HelloWorld },
    setup () {
      const msg = ref('Hello World')
      function toggle () {
        msg.value = msg.value === 'Hello World' ? 'Hello Vue' : 'Hello World'
      }
      return {
        toggle,
        msg
      }
    }
  }
</script>
```

可以看出 `setupContext` 对应的就是 `setup` 函数第二个参数，接下来看一下 `setup` 函数时如何执行的

```js
const setupResult = callWithErrorHandling(setup, instance, 0 /* SETUP_FUNCTION */, [instance.props, setupContext])

function callWithErrorHandling (fn, instance, type, args) {
  let res
  try {
    res = args ? fn(...args) : fn()
  }
  catch (err) {
    handleError(err, instance, type)
  }
  return res
}
```

`callWithErrorHandling` 对 `fn` 做了一层包装，内部执行 `fn`，所以 `setup` 函数的第一个参数是 `instance.props` 第二个参数是 `setupContext`

执行 `setup` 函数并拿到了返回的结果，那么接下来就是要用 `handleSetupResult` 函数来处理结果

```js
handleSetupResult(instance, setupResult)

function handleSetupResult(instance, setupResult) {
  if (isFunction(setupResult)) {
    // setup 返回渲染函数
    instance.render = setupResult
  }
  else if (isObject(setupResult)) {
    // 把 setup 返回结果变成响应式
    instance.setupState = reactive(setupResult)
  }
  finishComponentSetup(instance)
}
```

当 `setupResult` 是一个对象的时候，我们把它变成了响应式并赋值给 `instance.setupState`，这样在模板渲染的时候，依据前面的代理规则，`instance.ctx` 就可以从 `instance.setupState` 上获取到对应的数据，这就在 setup 函数与模板渲染间建立了联系

如果返回的是一个函数的话, 这个函数会作为组件的渲染函数

```js
<script>
  import { h } from 'vue'

  export default {
    props: {
      msg: String
    },
    setup (props, { emit }) {
      function onClick () {
        emit('toggle')
      }
      return (ctx) => {
        return [
          h('p', null, ctx.msg),
          h('button', { onClick: onClick }, 'Toggle')
        ]
      }
    }
  }
</script>
```

最后就是执行 finishComponentSetup 完成组件实例的设置

### 完成组件实例设置

```js
function finishComponentSetup (instance) {
  const Component = instance.type
  // 对模板或者渲染函数的标准化
  if (!instance.render) {
    if (compile && Component.template && !Component.render) {
      // 运行时编译
      Component.render = compile(Component.template, {
        isCustomElement: instance.appContext.config.isCustomElement || NO
      })
      Component.render._rc = true
    }
    if ((process.env.NODE_ENV !== 'production') && !Component.render) {
      if (!compile && Component.template) {
        // 只编写了 template 但使用了 runtime-only 的版本
        warn(`Component provided template option but ` +
          `runtime compilation is not supported in this build of Vue.` +
          (` Configure your bundler to alias "vue" to "vue/dist/vue.esm-bundler.js".`
          ) /* should not happen */)
      }
      else {
        // 既没有写 render 函数，也没有写 template 模板
        warn(`Component is missing template or render function.`)
      }
    }

    // 组件对象的 render 函数赋值给 instance
    instance.render = (Component.render || NOOP)
    if (instance.render._rc) {
      // 对于使用 with 块的运行时编译的渲染函数，使用新的渲染上下文的代理
      instance.withProxy = new Proxy(instance.ctx, RuntimeCompiledPublicInstanceProxyHandlers)
    }
  }
  // 兼容 Vue.js 2.x Options API
  {
    currentInstance = instance
    applyOptions(instance, Component)
    currentInstance = null
  }
}
```

`vue3` 依然支持 `vue2` 的 `options API`，这主要就是通过 `applyOptions` 方法实现的............................
