---
title: Traps in H5 in Wechat
date: 2016-12-08 00:24:19
tags:
---

1. `for (const foo of iterable) { ... }` 的语法在**安卓微信**中不能使用，哪怕用Babel进行转换也不行, 因为`iterable`会用到`Symbol.iterable`，而`Symbol`在安卓微信下是`undefined`的。
2. npm模块`react-css-module`需要使用`Object.assign`方法，虽然该方法有对应的polyfill，但是polyfill的行为和native不一样。使用该模块可能导致网页在低版本的浏览器中打不开。