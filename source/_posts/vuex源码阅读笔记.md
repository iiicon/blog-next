---
layout: post
title: vuex源码阅读笔记
date: 2020-09-14 15:36:42
tags: [vue, 笔记, vuex]
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
