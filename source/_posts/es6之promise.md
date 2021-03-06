---
title: es6之promise
date: 2018-11-05 19:48:35
tags: [es6]
categories: js
---

### promise 功能

  首先讲一下 promise 为什么会出现
    在实际的场景中，有非常多的场景我们不能立即知道该如何继续往下执行，最重要的就是ajax，
    通俗来说，由于网速的不同，可能你得到返回值的时间也是不同的，这个时候我们就需要等待，结果出来了之后才知道怎么样继续下去
    这个时候我们就需要用回调函数来执行，但是如果里面还有一层 ajax 请求，请求需要的新的参数还是由第一次 ajax 请求获得的
    当出现第三个ajax(甚至更多)仍然依赖上一个请求的时候，我们的代码就变成了一场灾难。这场灾难，往往也被称为回调地狱
    因此我们需要一个称为 promise 的东西来解决这个问题
    当然 promise 解决的不仅仅是回调地狱的问题，使代码具有更好的可读性和可维护性，可以将数据请求和数据处理分割开
    当然回调函数的原理呢就是函数调用栈
    promise 的原理呢就是 micro-task 队列
    如果浏览器已经支持了 promise 对象，那么我们就知道浏览器的 js 引擎里已经有了 promise 队列，这样我们就可以将
    我们的任务放到 promise 队列中去
    {
      1
      promise 有三种状态
      1 pending 等待中，进行中，表示还没有结果
      2 resolved(Fulfilled) 已经完成，得到了我们想要的结果，可以继续往下执行
      3 rejected 表示得到结果，但是不是想要的，所以拒绝执行
      这三种状态不受外界影响，而且状态只能从pending改变为resolved或者rejected，并且不可逆。
      在Promise对象的构造函数中，将一个函数作为第一个参数。而这个函数，就是用来处理Promise的状态变化。
      ```
      new Promise(function (resolve, reject) {
        if (true) { resolve() };
        if (false) { reject() };
      })
      ```
    }
    {
      2
      promise 对象中的 then 方法， 可以接受构造函数中处理的状态变化，并分别对应执行
      then 方法有 2 个参数， 第一个函数接受 resolved 状态的执行，第二个参数接受 reject 状态的执行
      ```
      function fn(num) {
        return new Promise(function (resolve, reject) {
          if (typeof num == 'number') {
            resolve();
          } else {
            reject();
          }
        }).then(function () {
          console.log('参数是一个number值');
        }, function () {
          console.log('参数不是一个number值');
        })
      }
      fn('hahha');
      fn(1234);
      ```
      then方法的执行结果也会返回一个Promise对象。因此我们可以进行then的链式执行，这也是解决回调地狱的主要方式。
    }
    {
      3 数据传递
      对 ajax 进行一个封装
      {
        var url = 'https://hq.tigerbrokers.com/fundamental/finance_calendar/getType/2017-02-26/2017-06-10';
        封装一个get请求的方法
        ```
        function getJSON(url) {
          return new Promise(function (resolve, reject) {
            var XHR = new XMLHttpRequest();
            XHR.open('GET', url, true);
            XHR.send();
            XHR.onreadystatechange = function () {
              if (XHR.readyState == 4) {
                if (XHR.status == 200) {
                  try {
                    var response = JSON.parse(XHR.responseText);
                    resolve(response);
                  } catch (e) {
                    reject(e);
                  }
                } else {
                  reject(new Error(XHR.statusText));
                }
              }
            }
          })
        }
        getJSON(url).then(resp => console.log(resp));
        window.getJSON = getJSON
        ```
      }
      总之，就是正确的结果就 resolve 一下， 错误的结果就 reject 一下， 
      并且利用上面的参数传递的方式，将正确的结果和错误的结果传递出来
    }
    {
      Promise.all
      当有一个 ajax 请求，他的参数需要另外两个甚至更多请求都有返回值之后才能确定，这个时候
      就需要 Promise.all 来帮我应对这个场景
      promise.all 接受一个 promise 数组作为参数，当这个数组的所有的 Promise 对象的状态都变成 resolved 和
      rejected 的时候，才会调用 then 方法
      ```
      var url = 'https://hq.tigerbrokers.com/fundamental/finance_calendar/getType/2017-02-26/2017-06-10';
      var url1 = 'https://hq.tigerbrokers.com/fundamental/finance_calendar/getType/2017-03-26/2017-06-10';
      function renderAll() {
        return Promise.all([getJSON(url), getJSON(url1)]);
      }
      renderAll().then(function (value) {
        建议大家在浏览器中看看这里的value值
        console.log(value);
      })
      ```
    }
    {
      promise.race
      与Promise.all相似的是，Promise.race都是以一个Promise对象组成的数组作为参数，
      不同的是，只要当数组中的其中一个Promsie状态变成resolved或者rejected时，就可以调用.then方法了。
      而传递给then方法的值也会有所不同，大家可以再浏览器中运行下面的例子与上面的例子进行对比
      {
        ```
        function renderRace() {
          return Promise.race([getJSON(url), getJSON(url1)]);
        }
        renderRace().then(function (value) {
          console.log(value);
        })
        ```
      }
    }

### promise 局限

1. 错误被吃掉

```
let promise = new Promise(() => {
    throw new Error('error')
});
console.log(2333333);
```
会正常的打印 233333，说明 Promise 内部的错误不会影响到 Promise 外部的代码，而这种情况我们就通常称为 “吃掉错误”

2. 无法取消

promise 一旦新建它就会立即执行，无法中途取消

3. 无法得知 pending 状态

当处于pending 状态时，无法得知目前进展到哪一个阶段

