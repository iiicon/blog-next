---
layout: post
title: Vue2数据驱动
date: 2020-09-22 09:37:11
tags: [vue2.x, 笔记]
categories: vue
---

Vue.js 的一个核心思想是数据驱动。所谓数据驱动，是指视图是由数据驱动生成的，我们对视图的修改，不会直接操作 DOM，而是通过修改数据。当交互复杂的时候，只关心数据的修改会让代码的逻辑变得非常清晰，因为 DOM 变成了数据的映射，我们所有的修改都是修改数据，而不用触碰 DOM，这样的代码非常利于维护。

<!-- more -->

## new Vue

Vue 定义：

```js
function Vue(options) {
  if (process.env.NODE_ENV !== "production" && !(this instanceof Vue)) {
    warn("Vue is a constructor and should be called with the `new` keyword");
  }
  this._init(options);
}
```

`Vue` 只能通过 `new` 调用，然后执行 `_init` 方法（这将是 Vue2 最重要的一个方法）

```js
Vue.prototype._init = function (options?: Object) {
  const vm: Component = this;
  // a uid
  vm._uid = uid++;

  let startTag, endTag;
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== "production" && config.performance && mark) {
    startTag = `vue-perf-start:${vm._uid}`;
    endTag = `vue-perf-end:${vm._uid}`;
    mark(startTag);
  }

  // a flag to avoid this being observed
  vm._isVue = true;
  // merge options
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options);
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    );
  }
  /* istanbul ignore else */
  if (process.env.NODE_ENV !== "production") {
    initProxy(vm);
  } else {
    vm._renderProxy = vm;
  }
  // expose real self
  vm._self = vm;
  initLifecycle(vm);
  initEvents(vm);
  initRender(vm);
  callHook(vm, "beforeCreate");
  initInjections(vm); // resolve injections before data/props
  initState(vm);
  initProvide(vm); // resolve provide after data/props
  callHook(vm, "created");

  /* istanbul ignore if */
  if (process.env.NODE_ENV !== "production" && config.performance && mark) {
    vm._name = formatComponentName(vm, false);
    mark(endTag);
    measure(`vue ${vm._name} init`, startTag, endTag);
  }

  if (vm.$options.el) {
    vm.$mount(vm.$options.el);
  }
};
```

`_init` 在实例化的时候就会调用，是 `Vue` 的核心，主要干了几件事情，合并配置，初始化事件中心，初始化渲染，初始化 data，初始化 props，初始化 computed，初始化 watcher 等等

下面列一下生命周期以及钩子的调用顺序，后面做剩余补充

```
init.js
- mergeOptions
- initLifecycle
- initEvents
- initRender
- callHook(vm, 'beforeCreate')
- initInjections
- initState
  - initProps
  - initMethods
  - initData
  - initComputed
  - initWatch
- initProvide
- callHook(vm, 'created')
```

## Vue 实例挂载的实现

`src/platform/web/entry-runtime-with-compiler.js` 中 `$mount` 的定义:

```js
const mount = Vue.prototype.$mount;
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el);

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== "production" &&
      warn(
        `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
      );
    return this;
  }

  const options = this.$options;
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template;
    if (template) {
      if (typeof template === "string") {
        if (template.charAt(0) === "#") {
          template = idToTemplate(template);
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== "production" && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            );
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML;
      } else {
        if (process.env.NODE_ENV !== "production") {
          warn("invalid template option:" + template, this);
        }
        return this;
      }
    } else if (el) {
      template = getOuterHTML(el);
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== "production" && config.performance && mark) {
        mark("compile");
      }

      const { render, staticRenderFns } = compileToFunctions(
        template,
        {
          shouldDecodeNewlines,
          shouldDecodeNewlinesForHref,
          delimiters: options.delimiters,
          comments: options.comments,
        },
        this
      );
      options.render = render;
      options.staticRenderFns = staticRenderFns;

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== "production" && config.performance && mark) {
        mark("compile end");
        measure(`vue ${this._name} compile`, "compile", "compile end");
      }
    }
  }
  return mount.call(this, el, hydrating);
};
```

这里首先缓存了原型上的 `$mount` 方法，首先对 `el` 做了限制，`Vue` 不能挂载在 `body html` 这样的根节点上，如果没有定义 `render` 方法，~~则会把 `el` 或者 `template` 字符串转换成 `render` 方法。~~
这里我们要牢记，`Vue2` 版本中，所有 `Vue` 的组件的渲染最终都需要 `render` 方法，无论我们是用单文件 `.vue` 的方式开发组件还是写了 `el` 或者 `template` 字符串转换成 `render` 方法，那么这么过程是 `Vue` 的一个在线编译的过程，它是调用 `compileToFunctions` 方法实现的

```js
// public mount method
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined;
  return mountComponent(this, el, hydrating);
};
```

公共的 `$mount` 方法传入两个参数，第一个是 `el`，它表示挂载的元素，可以是字符串，也可以是 `DOM` 对象，第二个是否是服务端渲染，`$mount` 方法实际上会去调用 `mountComponent` 方法

```js
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el;
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode;
    if (process.env.NODE_ENV !== "production") {
      /* istanbul ignore if */
      if (
        (vm.$options.template && vm.$options.template.charAt(0) !== "#") ||
        vm.$options.el ||
        el
      ) {
        warn(
          "You are using the runtime-only build of Vue where the template " +
            "compiler is not available. Either pre-compile the templates into " +
            "render functions, or use the compiler-included build.",
          vm
        );
      } else {
        warn(
          "Failed to mount component: template or render function not defined.",
          vm
        );
      }
    }
  }
  callHook(vm, "beforeMount");

  let updateComponent;
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== "production" && config.performance && mark) {
    updateComponent = () => {
      const name = vm._name;
      const id = vm._uid;
      const startTag = `vue-perf-start:${id}`;
      const endTag = `vue-perf-end:${id}`;

      mark(startTag);
      const vnode = vm._render();
      mark(endTag);
      measure(`vue ${name} render`, startTag, endTag);

      mark(startTag);
      vm._update(vnode, hydrating);
      mark(endTag);
      measure(`vue ${name} patch`, startTag, endTag);
    };
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating);
    };
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(
    vm,
    updateComponent,
    noop,
    {
      before() {
        if (vm._isMounted) {
          callHook(vm, "beforeUpdate");
        }
      },
    },
    true /* isRenderWatcher */
  );
  hydrating = false;

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true;
    callHook(vm, "mounted");
  }
  return vm;
}
```

`mountComponent` 核心就是实例化一个渲染 `Watcher`，在它的回调函数中调用 `updateComponent` 方法，在此方法中调用 `vm._render` 方法生成虚拟 `Node`，最终调用 `vm._update` 更新 `DOM`
这里的 watcher 有两个作用，一个是初始化的时候会执行回调函数，另一个是当 vm 实例中的监测数据发生变化的时候执行回调函数，这里第四个参数是 watcher 构造函数的 options 参数之一，执行 `this.before = options.before` 保留了 `before` 钩子函数

函数最后判断为根节点的时候设置 `vm._isMounte`d 为 `true`，表示这个实例已经挂载了，同时执行 `mounted` 钩子函数。这里注意 `vm.$vnode` 表示 `Vue` 实例的父虚拟 `Node`，它是 `null` 表示是根 Vue 的实例

这里也涉及到了两个钩子函数

```
new watcher(vm, updateComponent, noop, {beforeUpdate})
updateComponent
callHook(vm, 'mounted')
```

## render

`Vue` 的 `_render` 方法是实例的一个私有方法，它用来把实例渲染成一个虚拟 `Node`

```js
Vue.prototype._render = function (): VNode {
  const vm: Component = this;
  const { render, _parentVnode } = vm.$options;

  // reset _rendered flag on slots for duplicate slot check
  if (process.env.NODE_ENV !== "production") {
    for (const key in vm.$slots) {
      // $flow-disable-line
      vm.$slots[key]._rendered = false;
    }
  }

  if (_parentVnode) {
    vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject;
  }

  // set parent vnode. this allows render functions to have access
  // to the data on the placeholder node.
  vm.$vnode = _parentVnode;
  // render self
  let vnode;
  try {
    vnode = render.call(vm._renderProxy, vm.$createElement);
  } catch (e) {
    handleError(e, vm, `render`);
    // return error render result,
    // or previous vnode to prevent render error causing blank component
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== "production") {
      if (vm.$options.renderError) {
        try {
          vnode = vm.$options.renderError.call(
            vm._renderProxy,
            vm.$createElement,
            e
          );
        } catch (e) {
          handleError(e, vm, `renderError`);
          vnode = vm._vnode;
        }
      } else {
        vnode = vm._vnode;
      }
    } else {
      vnode = vm._vnode;
    }
  }
  // return empty vnode in case the render function errored out
  if (!(vnode instanceof VNode)) {
    if (process.env.NODE_ENV !== "production" && Array.isArray(vnode)) {
      warn(
        "Multiple root nodes returned from render function. Render function " +
          "should return a single root node.",
        vm
      );
    }
    vnode = createEmptyVNode();
  }
  // set parent
  vnode.parent = _parentVnode;
  return vnode;
};
```

这段代码最关键的是 `render` 方法调用，我们在平时的开发工作中手写 `render` 方法的场景比较少，而写的比较多的是 `template` 模板，在之前的 `compileToFunctions` 方法的实现中，会把 template 编译成 `render` 方法，但这个编译过程是非常复杂的

我们知道 `render` 方法的第一个参数是 `createElement`

```js
<div id="app">{{ message }}</div>
```

相当于我们编写的 render 函数

```js
render: function (createElement) {
  return createElement('div', {
     attrs: {
        id: 'app'
      },
  }, this.message)
}
```

再回到 `_render` 函数中的 `render` 方法的调用

```js
vnode = render.call(vm._renderProxy, vm.$createElement);
```

可以看到，`render` 函数中的 `createElement` 方法就是 `vm.$createElement` 方法

```js
export function initRender(vm: Component) {
  // ...
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false);
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true);
}
```

实际上, `vm.$createElement` 方法的定义是在执行 `initRender` 方法的时候，可以看到还有一个` vm._c` 方法，它是被模板编译成的 `render` 函数调用，`vm.$createElement` 是用户手写 `render` 方法调用两个都调用 `createElement`，最后一个参数其实就是递归拍平 `children vnode`

 
## Virtual DOM

虚拟 DOM 是在浏览器的真实 DOM 的前提下产生的，它产生的前提是 DOM 是很昂贵的，dom元素上面的属性非常多，频繁的更新它就会有性能上的问题

Virtural DOM 是用一个原生的 JS 对象去描述一个 DOM 节点，所以它比创建一个 DOM 的代价要小的多。在 Vue 中，Virtual DOM 是用 VNode 这么一个 class 去描述

```js
export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  fnScopeId: ?string; // functional scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.fnContext = undefined
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}
```

VNode 是对真实 DOM 的一种抽象描述，它的定义无非就几个关键属性，标签名，数据，子节点，键值等等，其他属性都是用扩展 VNode 的，它只是有数据结构的定义，映射到真实的 DOM 实际上要经历 VNode 的 create diff patch 等过程。那么在 Vue.js 中，VNode 是通过之前提到的 createElement 方法创建的

## 