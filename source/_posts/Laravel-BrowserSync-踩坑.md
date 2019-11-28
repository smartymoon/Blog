---
title: Laravel BrowserSync 踩坑
date: 2018-10-26 15:55:22
tags:
---

在 laravel 开发中，mix 是一个非常好用的工具。为了少按几次F5，我们可以使用 BrowserSync,官方文档如下
```
mix.browserSync('my-domain.test');

// Or...

// https://browsersync.io/docs/options
mix.browserSync({
    proxy: 'my-domain.test'
});
```
坑来了，千万不要在 Homestead 中开启这个服务，而是要在真机中开启动这个服务，否则会造成文件无法监听修改的问题
