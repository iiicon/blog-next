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
