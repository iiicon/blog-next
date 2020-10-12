---
layout: post
title: js最长递增子序列
date: 2020-10-12 20:04:50
tags: [code, 算法]
categories: js
---

## vue3 DOM diff 最长递增子序列的实现

```js
// https://en.wikipedia.org/wiki/Longest_increasing_subsequence
function getSequence(arr: number[]): number[] {
  // 数组副本
  const p = arr.slice();
  // 最后返回的序列 存储的是长度为 i 的递增子序列最小末尾值的索引
  const result = [0];

  let i, j, u, v, c;
  const len = arr.length;

  for (i = 0; i < len; i++) {
    // 保存数组第 i 项
    const arrI = arr[i];

    if (arrI !== 0) {
      // 取出现有的列表最后一个值
      j = result[result.length - 1];

      // 如果第 i-1 项小于第 i 项，就要把下标 i 排列到 result 的最后，继续下一个循环
      if (arr[j] < arrI) {
        p[i] = j;
        result.push(i);
        continue;
      }

      // 否则就要挨个和前面的比较
      u = 0; // 索引
      // 保存现有 result 的长度
      v = result.length - 1;

      // 利用二分查找出比 第 i 项小的值(最小差，也就是最接近) c，并把 result 的长度设为 c
      while (u < v) {
        c = ((u + v) / 2) | 0;
        if (arr[result[c]] < arrI) {
          u = c + 1;
        } else {
          v = c;
        }
      }


      if (arrI < arr[result[u]]) {
        if (u > 0) {
          p[i] = result[u - 1];
        }
        result[u] = i;
      }
    }
  }

  u = result.length;
  v = result[u - 1];
  
  while (u-- > 0) {
    result[u] = v;
    v = p[v];
  }
  return result;
}
```
