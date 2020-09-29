---
title: vue3 源码阅读笔记
date: 2020-08-20 10:03:35
tags: [vue3, 笔记]
categories: vue
---

# Vue.js 的优化

Vue 3.0 从源码、性能和语法 API 三个大的方面优化了框架，也提高了开发人员的开发体验，相比于 2.x 有了很大的一个改变

<!-- more -->

## 源码优化

- 更好的代码管理方式：monorepo
- 有类型的 JavaScript：TypeScript

## 性能优化

- 源码体积优化

  - 首先，移除一些冷门的 feature（比如 filter、inline-template 等）
  - 其次，引入 tree-shaking 的技术，减少打包体积

- 数据劫持优化
- 编译优化，借助 Block tree，Vue.js 将 vnode 更新性能由与模版整体大小相关提升为与动态内容的数量相关，这是一个非常大的性能突破

## 语法 API 优化：Composition API

1. 优化逻辑组织
2. 优化逻辑复用

## 引入 RFC：使每个版本改动可控

[Vue.js-RFC](https://github.com/vuejs/rfcs/pulls?q=is%3Apr+is%3Amerged+label%3A3.x)

# 组件的实现：Vue 核心的实现

> 模板 + 对象描述 + 数据 = 组件

## vnode 到真实 DOM 是如何转变的？

> 创建 vnode + 渲染 vnode + 生成 DOM

### vue 初始化

```js
// 在 Vue.js 3.0 中，初始化一个应用的方式如下
import { createApp } from "vue";
import App from "./app";
const app = createApp(App);
app.mount("#app"); // 把 App 组件挂载到 id 为 app 的 DOM 节点上
```

这其中导入了一个 createApp 入口函数，他是 Vue.js 对外暴露的一个函数

```js
const createApp = (...args) => {
  // 创建 app 对象
  const app = ensureRenderer().createApp(...args);
  const { mount } = app;
  // 重写 mount 方法
  app.mount = (containerOrSelector) => {
    // ...
  };
  return app;
};
```

从代码中可以看出 createApp 主要做了两件事情，创建 app 对象和重写 app.mount 方法

#### 1. 创建 app 对象

```js
const app = ensureRenderer().createApp(...args);
```

其中 ensureRenderer() 用来创建一个渲染器对象，它的内部实现如下

```js
// 渲染相关的一些配置，比如更新属性的方法，操作 DOM 的方法
const rendererOptions = {
  patchProp,
  ...nodeOps,
};
let renderer;
// 延时创建渲染器，当用户只依赖响应式包的时候，可以通过 tree-shaking 移除核心渲染逻辑相关的代码
function ensureRenderer() {
  return renderer || (renderer = createRenderer(rendererOptions));
}
function createRenderer(options) {
  return baseCreateRenderer(options);
}
function baseCreateRenderer(options) {
  function render(vnode, container) {
    // 组件渲染的核心逻辑
  }

  return {
    render,
    createApp: createAppAPI(render),
  };
}
function createAppAPI(render) {
  // createApp createApp 方法接受的两个参数：根组件的对象和 prop
  return function createApp(rootComponent, rootProps = null) {
    const app = {
      _component: rootComponent,
      _props: rootProps,
      mount(rootContainer) {
        // 创建根组件的 vnode
        const vnode = createVNode(rootComponent, rootProps);
        // 利用渲染器渲染 vnode
        render(vnode, rootContainer);
        app._container = rootContainer;
        return vnode.component.proxy;
      },
    };
    return app;
  };
}
```

#### 2. 重写 app.mount 方法

Vue.js 不仅仅是为 Web 平台服务，它的目标是支持跨平台渲染，而 createApp 函数内部的 app.mount 方法是一个标准的可跨平台的组件渲染流程，

```js
mount(rootContainer) {
 // 创建根组件的 vnode
 const vnode = createVNode(rootComponent, rootProps)
 // 利用渲染器渲染 vnode
 render(vnode, rootContainer)
 app._container = rootContainer
 return vnode.component.proxy
}
```

标准的跨平台渲染流程是先创建 vnode，再渲染 vnode。此外参数 rootContainer 也可以是不同类型的值，也就是这里是通用的渲染逻辑，
接下来完善 Web 平台下的渲染逻辑

```js
app.mount = (containerOrSelector) => {
  // 标准化容器
  const container = normalizeContainer(containerOrSelector);
  if (!container) return;
  const component = app._component;
  // 如组件对象没有定义 render 函数和 template 模板，则取容器的 innerHTML 作为组件模板内容
  if (!isFunction(component) && !component.render && !component.template) {
    component.template = container.innerHTML;
  }
  // 挂载前清空容器内容
  container.innerHTML = "";
  // 真正的挂载
  return mount(container);
};
```

app.mount 就是 重写的 mount 方法，传入 container 参数，先标准化容器，然后取出 rootComponent，
如组件对象没有定义 render 函数和 template 模板，则取容器的 innerHTML 作为组件模板内容，
在挂载前清空容器内容，然后执行通用的 mount 方法

### 核心渲染流程：创建 vnode 和渲染 vnode

#### 创建 vnode

组件 vnode 其实是对抽象事物的描述，这是因为我们并不会在页面上真正渲染一个 `<custom-component>` 标签，而是渲染组件内部定义的 HTML 标签。
vnode 有组件 vnode，普通元素 vnode，注释 vnode，文本 vnode

**为什么要设计 vnode？**

- 抽象
- 跨平台都可以用

回顾 app.mount 内部实现，用 createVnode 创建了根组件的 vnode

```js
const vnode = createVNode(rootComponent, rootProps);
```

```js
function createVNode(type, props = null, children = null) {
  if (props) {
    // 处理 props 相关逻辑，标准化 class 和 style
  }
  // 对 vnode 类型信息编码
  const shapeFlag = isString(type)
    ? 1 /* ELEMENT */
    : isSuspense(type)
    ? 128 /* SUSPENSE */
    : isTeleport(type)
    ? 64 /* TELEPORT */
    : isObject(type)
    ? 4 /* STATEFUL_COMPONENT */
    : isFunction(type)
    ? 2 /* FUNCTIONAL_COMPONENT */
    : 0;
  const vnode = {
    type,
    props,
    shapeFlag,
    // 一些其他属性
  };
  // 标准化子节点，把不同数据类型的 children 转成数组或者文本类型
  normalizeChildren(vnode, children);
  return vnode;
}
```

可以看出 createVnode 就是对 props 做标准化处理、对 vnode 的类型信息编码、创建 vnode 对象，标准化子节点 children，返回 vnode

#### 渲染 vnode

在 app.mount 内部通过 render(vnode, rootContainer) 去渲染组件

```js
render(vnode, rootContainer);
const render = (vnode, container) => {
  if (vnode == null) {
    // 销毁组件
    if (container._vnode) {
      unmount(container._vnode, null, null, true);
    }
  } else {
    // 创建或者更新组件
    patch(container._vnode || null, vnode, container);
  }
  // 缓存 vnode 节点，表示已经渲染
  container._vnode = vnode;
};
```

通过 vnode 判断去执行卸载还是创建或更新，接下来看 patch

```js
const patch = (
  n1,
  n2,
  container,
  anchor = null,
  parentComponent = null,
  parentSuspense = null,
  isSVG = false,
  optimized = false
) => {
  // 如果存在新旧节点, 且新旧节点类型不同，则销毁旧节点
  if (n1 && !isSameVNodeType(n1, n2)) {
    anchor = getNextHostNode(n1);
    unmount(n1, parentComponent, parentSuspense, true);
    n1 = null;
  }
  const { type, shapeFlag } = n2;
  switch (type) {
    case Text:
      // 处理文本节点
      break;
    case Comment:
      // 处理注释节点
      break;
    case Static:
      // 处理静态节点
      break;
    case Fragment:
      // 处理 Fragment 元素
      break;
    default:
      if (shapeFlag & 1 /* ELEMENT */) {
        // 处理普通 DOM 元素
        processElement(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        );
      } else if (shapeFlag & 6 /* COMPONENT */) {
        // 处理组件
        processComponent(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        );
      } else if (shapeFlag & 64 /* TELEPORT */) {
        // 处理 TELEPORT
      } else if (shapeFlag & 128 /* SUSPENSE */) {
        // 处理 SUSPENSE
      }
  }
};
```

我们重点关注对组件的处理和对普通 dom 元素的处理
先看对组件的处理

```js
const processComponent = (
  n1,
  n2,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized
) => {
  if (n1 == null) {
    // 挂载组件
    mountComponent(
      n2,
      container,
      anchor,
      parentComponent,
      parentSuspense,
      isSVG,
      optimized
    );
  } else {
    // 更新组件
    updateComponent(n1, n2, parentComponent, optimized);
  }
};
```

mountComponent 就做三件事情

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

组件的创建不像 2.0 去实例化组件，内部通过返回对象创建，接着是设置组件实例，保留了很多组件相关的数据，维护了组件的上下文，包括对 props，插槽，以及其他实例的属性的初始化处理，最后是运行带副作用的渲染函数 `setupRenderEffect`

```js
const setupRenderEffect = (
  instance,
  initialVNode,
  container,
  anchor,
  parentSuspense,
  isSVG,
  optimized
) => {
  // 创建响应式的副作用渲染函数
  instance.update = effect(function componentEffect() {
    if (!instance.isMounted) {
      // 渲染组件生成子树 vnode
      const subTree = (instance.subTree = renderComponentRoot(instance));
      // 把子树 vnode 挂载到 container 中
      patch(null, subTree, container, anchor, instance, parentSuspense, isSVG);
      // 保留渲染生成的子树根 DOM 节点
      initialVNode.el = subTree.el;
      instance.isMounted = true;
    } else {
      // 更新组件
    }
  }, prodEffectOptions);
};
```

该函数利用响应式库的 `effect` 函数创建了一个副作用渲染函数 `componentEffect`，当组件的数据发生变化时，`effect` 函数包裹的内部渲染函数 `componentEffect` 会重新执行一遍，从而达到重新渲染组件的目的。

先分析初始渲染

**初始渲染主要做两件事情：渲染组件生成 subTree、把 subTree 挂载到 container 中**

`initialVnode` 就是 `组件 vnode`，`subTree` 就是 `子树 vnode` 执行 `renderComponentRoot` 生成

我们知道每个组件都会有对应的 `render` 函数，即使你写 `template`，也会编译成 `render` 函数，而 `renderComponentRoot` 函数就是去执行 `render` 函数创建整个组件树内部的 `vnode`，把这个 `vnode` 再经过内部一层标准化，就得到了该函数的返回结果：子树 vnode。

渲染成子树 `vnode` 后，接下来就是继续调用 `patch` 函数把子树 `vnode` 挂载到 `container` 中

又回到 `patch` 函数，会继续对这个子树的 `vnode` 类型进行判断，如果是 `div` 就对应的是 `普通元素 vnode`，
就会处理普通 dom 执行 `processElement` 函数

```js
const processElement = (
  n1,
  n2,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized
) => {
  isSVG = isSVG || n2.type === "svg";
  if (n1 == null) {
    //挂载元素节点
    mountElement(
      n2,
      container,
      anchor,
      parentComponent,
      parentSuspense,
      isSVG,
      optimized
    );
  } else {
    //更新元素节点
    patchElement(n1, n2, parentComponent, parentSuspense, isSVG, optimized);
  }
};
```

接下来看 mountElement 的逻辑

```js
const mountElement = (
  vnode,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized
) => {
  let el;
  const { type, props, shapeFlag } = vnode;
  // 创建 DOM 元素节点
  el = vnode.el = hostCreateElement(vnode.type, isSVG, props && props.is);
  if (props) {
    // 处理 props，比如 class、style、event 等属性
    for (const key in props) {
      if (!isReservedProp(key)) {
        hostPatchProp(el, key, null, props[key], isSVG);
      }
    }
  }
  if (shapeFlag & 8 /* TEXT_CHILDREN */) {
    // 处理子节点是纯文本的情况
    hostSetElementText(el, vnode.children);
  } else if (shapeFlag & 16 /* ARRAY_CHILDREN */) {
    // 处理子节点是数组的情况 循环执行 child patch
    mountChildren(
      vnode.children,
      el,
      null,
      parentComponent,
      parentSuspense,
      isSVG && type !== "foreignObject",
      optimized || !!vnode.dynamicChildren
    );
  }
  // 把创建的 DOM 元素节点挂载到 container 上
  hostInsert(el, container, anchor);
};
```

可以看到，挂载元素函数主要做四件事情，创建 `DOM` 元素节点，处理 `props`，处理 `children`，挂载 `DOM` 到 container 上

渲染完 `subTree` 之后，就会执行 `componentEffect` 函数中的剩余逻辑

```js
initialVNode.el = subTree.el;
instance.isMounted = true;
```

## 组件更新：完整的 DOM diff 流程是怎样的？（上）

### 副作用渲染函数更新组件的过程

带副作用的渲染函数 `setupRenderEffect` 会在 instance.update 上挂载 `componentEffect` 函数，数据变化后就会执行该函数的 `else` 逻辑

```js
const setupRenderEffect = (
  instance,
  initialVNode,
  container,
  anchor,
  parentSuspense,
  isSVG,
  optimized
) => {
  // 创建响应式的副作用渲染函数
  instance.update = effect(function componentEffect() {
    if (!instance.isMounted) {
      // 渲染组件
    } else {
      // 更新组件
      let { next, vnode } = instance;
      // next 表示新的组件 vnode
      if (next) {
        // 更新组件 vnode 节点信息
        updateComponentPreRender(instance, next, optimized);
      } else {
        next = vnode;
      }
      // 渲染新的子树 vnode
      const nextTree = renderComponentRoot(instance);
      // 缓存旧的子树 vnode
      const prevTree = instance.subTree;
      // 更新子树 vnode
      instance.subTree = nextTree;
      // 组件更新核心逻辑，根据新旧子树 vnode 做 patch
      patch(
        prevTree,
        nextTree,
        // 如果在 teleport 组件中父节点可能已经改变，所以容器直接找旧树 DOM 元素的父节点
        hostParentNode(prevTree.el),
        // 参考节点在 fragment 的情况可能改变，所以直接找旧树 DOM 元素的下一个节点
        getNextHostNode(prevTree),
        instance,
        parentSuspense,
        isSVG
      );
      // 缓存更新后的 DOM 节点
      next.el = nextTree.el;
    }
  }, prodEffectOptions);
};
```

可以看到，更新组件主要做三件事情：更新组件 `vnode` 节点，渲染新的子树 `vnode` 根据新旧子树 `vnode` 执行 `patch` 逻辑

更新组件 `vnode` 的时候，要判断有没有新组件 `vnode` `next`，有则更新，没有就用之前的 `vnode`
渲染新的子树 `vnode` 和之前的一样
最后就是核心的 `patch` 逻辑，用来找出新旧子树 vnode 的不同，并找到一种合适的方式更新 DOM

### 核心 patch

```js
const patch = (
  n1,
  n2,
  container,
  anchor = null,
  parentComponent = null,
  parentSuspense = null,
  isSVG = false,
  optimized = false
) => {
  // 如果存在新旧节点, 且新旧节点类型不同，则销毁旧节点
  if (n1 && !isSameVNodeType(n1, n2)) {
    anchor = getNextHostNode(n1);
    unmount(n1, parentComponent, parentSuspense, true);
    // n1 设置为 null 保证后续都走 mount 逻辑
    n1 = null;
  }
  const { type, shapeFlag } = n2;
  switch (type) {
    case Text:
      // 处理文本节点
      break;
    case Comment:
      // 处理注释节点
      break;
    case Static:
      // 处理静态节点
      break;
    case Fragment:
      // 处理 Fragment 元素
      break;
    default:
      if (shapeFlag & 1 /* ELEMENT */) {
        // 处理普通 DOM 元素
        processElement(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        );
      } else if (shapeFlag & 6 /* COMPONENT */) {
        // 处理组件
        processComponent(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        );
      } else if (shapeFlag & 64 /* TELEPORT */) {
        // 处理 TELEPORT
      } else if (shapeFlag & 128 /* SUSPENSE */) {
        // 处理 SUSPENSE
      }
  }
};
function isSameVNodeType(n1, n2) {
  // n1 和 n2 节点的 type 和 key 都相同，才是相同节点
  return n1.type === n2.type && n1.key === n2.key;
}
```

在这个过程中，首先判断新旧节点是否是相同的 `vnode` 类型，如果不同，比如一个 `div` 更新成一个 `ul`，那么最简单的操作就是删除旧的 `div` 节点，再去挂载新的 `ul` 节点

如果是相同的 `vnode`，那就要走 `diff` 更新流程了，接着会根据不同的 `vnode` 类型执行不同的处理逻辑，这里我们仍然只分析普通元素类型和组件类型的处理过程

#### 处理组件

如何处理组件的呢？举个例子，我们在父组件 App 中里引入了 Hello 组件

```js
<template>
  <div class="app">
    <p>This is an app.</p>
    <hello :msg="msg"></hello>
    <button @click="toggle">Toggle msg</button>
  </div>
</template>
<script>
  export default {
    data() {
      return {
        msg: 'Vue'
      }
    },
    methods: {
      toggle() {
        this.msg = this.msg ==== 'Vue'? 'World': 'Vue'
      }
    }
  }
</script>

<template>
  <div class="hello">
    <p>Hello, {{msg}}</p>
  </div>
</template>
<script>
  export default {
    props: {
      msg: String
    }
  }
</script>
```

点击 App 组件中的按钮执行 toggle 函数，就会修改 data 中的 msg，并且会触发 App 组件的重新渲染

结合前面对渲染流程的分析，这里的 App 组件的根节点是 div，重新渲染的子树 vnode 节点是一个普通元素的 vnode，应该先走 processElement 逻辑，组件的更新最终还是要转换成真实的 DOM 更新
而实际上普通元素的处理才是 DOM 的更新
和渲染流程相似，更新过程也是一个树的深度优先遍历的过程，更新完当前节点后，就会遍历更新它的子节点，因此在遍历的过程中会遇到 hello 这个 组件 vnode 节点，就会执行到 processComponent 的处理

```js
const processComponent = (
  n1,
  n2,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized
) => {
  if (n1 == null) {
    // 挂载组件
  } else {
    // 更新子组件
    updateComponent(n1, n2, parentComponent, optimized);
  }
};
const updateComponent = (n1, n2, parentComponent, optimized) => {
  const instance = (n2.component = n1.component);
  // 根据新旧子组件 vnode 判断是否需要更新子组件
  if (shouldUpdateComponent(n1, n2, parentComponent, optimized)) {
    // 新的子组件 vnode 赋值给 instance.next
    instance.next = n2;
    // 子组件也可能因为数据变化被添加到更新队列里了，移除它们防止对一个子组件重复更新
    invalidateJob(instance.update);
    // 执行子组件的副作用渲染函数
    instance.update();
  } else {
    // 不需要更新，只复制属性
    n2.component = n1.component;
    n2.el = n1.el;
  }
};
```

可以看到， `processComponent` 通过 `updateComponent` 函数来更新组件，`updateComponent` 函数在更新子组件的时候，会先执行 `shouldUpdateComponent` 函数，根据新旧子组件 `vnode` 来判断是否需要更新子组件
如果 `shouldUpdateComponent` 返回 true， 那么在它的最后先执行` invalidateJob（instance.update）`避免子组件由于自身数据变化导致的重复更新，然后又执行了子组件的副作用渲染函数 `instance.update` 来主动触发子组件的更新

然后再到 `setupRenderEffect` 函数的 `update` 逻辑

```js
// 更新组件
let { next, vnode } = instance;
// next 表示新的组件 vnode
if (next) {
  // 更新组件 vnode 节点信息
  updateComponentPreRender(instance, next, optimized);
} else {
  next = vnode;
}
const updateComponentPreRender = (instance, nextVNode, optimized) => {
  // 新组件 vnode 的 component 属性指向组件实例
  nextVNode.component = instance;
  // 旧组件 vnode 的 props 属性
  const prevProps = instance.vnode.props;
  // 组件实例的 vnode 属性指向新的组件 vnode
  instance.vnode = nextVNode;
  // 清空 next 属性，为了下一次重新渲染准备
  instance.next = null;
  // 更新 props
  updateProps(instance, nextVNode.props, prevProps, optimized);
  // 更新 插槽
  updateSlots(instance, nextVNode.children);
};
```

结合上面的代码，我们在更新组件的 `DOM` 前，需要先更新组件 `vnode` 节点信息，包括更改组件实例的 `vnode` 指针、更新 `props` 和更新插槽等一系列操作，因为组件在稍后执行 `renderComponentRoot` 时会重新渲染新的子树 `vnode` ，它依赖了更新后的组件 `vnode` 中的 `props` 和 `slots` 等数据。

所以我们现在知道了一个组件重新渲染可能会有两种场景，一种是组件本身的数据变化，这种情况下 `next` 是 `null`；另一种是父组件在更新的过程中，遇到子组件节点，先判断子组件是否需要更新，如果需要则主动执行子组件的重新渲染方法，这种情况下 `next` 就是新的子组件 `vnode`

所以 `processComponent` 处理组件 `vnode`，本质上就是去判断子组件是否需要更新，如果需要则递归执行子组件的副作用渲染函数来更新，否则仅仅更新一些 `vnode` 的属性，并让子组件实例保留对组件 `vnode` 的引用，用于子组件自身数据变化引起组件重新渲染的时候，在渲染函数内部可以拿到新的组件 `vnode`

前面也说过，组件是抽象的，组件的更新最终还是会落到对普通 `DOM` 元素的更新。所以接下来我们详细分析一下组件更新中对普通元素的处理流程

#### 处理普通元素

```js
const processElement = (
  n1,
  n2,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized
) => {
  isSVG = isSVG || n2.type === "svg";
  if (n1 == null) {
    // 挂载元素
  } else {
    // 更新元素
    patchElement(n1, n2, parentComponent, parentSuspense, isSVG, optimized);
  }
};
const patchElement = (
  n1,
  n2,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized
) => {
  const el = (n2.el = n1.el);
  const oldProps = (n1 && n1.props) || EMPTY_OBJ;
  const newProps = n2.props || EMPTY_OBJ;
  // 更新 props
  patchProps(
    el,
    n2,
    oldProps,
    newProps,
    parentComponent,
    parentSuspense,
    isSVG
  );
  const areChildrenSVG = isSVG && n2.type !== "foreignObject";
  // 更新子节点
  patchChildren(
    n1,
    n2,
    el,
    null,
    parentComponent,
    parentSuspense,
    areChildrenSVG
  );
};
```

可以看到，更新元素的过程主要做两件事情：更新 `props` 和更新子节点。其实这是很好理解的，因为一个 `DOM` 节点元素就是由它自身的一些属性和子节点构成的。

首先是更新 `props`，这里的 `patchProps` 函数就是在更新 `DOM` 节点的 `class、style、event` 以及其它的一些 `DOM` 属性

其次是更新子节点，我们来看一下这里的 `patchChildren` 函数的实现：

```js
const patchChildren = (
  n1,
  n2,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized = false
) => {
  const c1 = n1 && n1.children;
  const prevShapeFlag = n1 ? n1.shapeFlag : 0;
  const c2 = n2.children;
  const { shapeFlag } = n2;
  // 子节点有 3 种可能情况：文本、数组、空
  if (shapeFlag & 8 /* TEXT_CHILDREN */) {
    if (prevShapeFlag & 16 /* ARRAY_CHILDREN */) {
      // 数组 -> 文本，则删除之前的子节点
      unmountChildren(c1, parentComponent, parentSuspense);
    }
    if (c2 !== c1) {
      // 文本对比不同，则替换为新文本
      hostSetElementText(container, c2);
    }
  } else {
    if (prevShapeFlag & 16 /* ARRAY_CHILDREN */) {
      // 之前的子节点是数组
      if (shapeFlag & 16 /* ARRAY_CHILDREN */) {
        // 新的子节点仍然是数组，则做完整地 diff
        patchKeyedChildren(
          c1,
          c2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        );
      } else {
        // 数组 -> 空，则仅仅删除之前的子节点
        unmountChildren(c1, parentComponent, parentSuspense, true);
      }
    } else {
      // 之前的子节点是文本节点或者为空
      // 新的子节点是数组或者为空
      if (prevShapeFlag & 8 /* TEXT_CHILDREN */) {
        // 如果之前子节点是文本，则把它清空
        hostSetElementText(container, "");
      }
      if (shapeFlag & 16 /* ARRAY_CHILDREN */) {
        // 如果新的子节点是数组，则挂载新子节点
        mountChildren(
          c2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        );
      }
    }
  }
};
```

**对于一个元素的子节点 vnode 可能有三种情况：纯文本、vnode 数据 和 空。所以排列组合更新的时候就有 9 种情况**

1. 首先来看一下旧子节点是纯文本的情况：

   - 如果新子节点也是纯文本，那么做简单地文本替换即可；

   - 如果新子节点是空，那么删除旧子节点即可；

   - 如果新子节点是 vnode 数组，那么先把旧子节点的文本清空，再去旧子节点的父容器下添加多个新子节点。

2. 接下来看一下旧子节点是空的情况：

   - 如果新子节点是纯文本，那么在旧子节点的父容器下添加新文本节点即可；

   - 如果新子节点也是空，那么什么都不需要做；

   - 如果新子节点是 vnode 数组，那么直接去旧子节点的父容器下添加多个新子节点即可。

3. 最后来看一下旧子节点是 vnode 数组的情况：

   - 如果新子节点是纯文本，那么先删除旧子节点，再去旧子节点的父容器下添加新文本节点；

   - 如果新子节点是空，那么删除旧子节点即可；

   - 如果新子节点也是 vnode 数组，那么就需要做完整的 diff 新旧子节点了，这是最复杂的情况，内部运用了核心 diff 算法。

## 问题

- 虽然 Vue.js 的更新粒度是组件级别的，组件的数据变化只会影响当前组件的更新，但是在组件更新的过程中，也会对子组件做一定的检查，判断子组件是否也要更新，并通过某种机制避免子组件重复更新。
- processComponent 处理组件 vnode，本质上就是去判断子组件是否需要更新，如果需要则递归执行子组件的副作用渲染函数来更新，否则仅仅更新一些 vnode 的属性，并让子组件实例保留对组件 vnode 的引用，用于子组件自身数据变化引起组件重新渲染的时候，在渲染函数内部可以拿到新的组件 vnode。
