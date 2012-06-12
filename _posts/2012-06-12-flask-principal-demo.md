---
layout: post
title: "flask-principal使用小记"
description: "使用flask-principal来对基于flask的web应用进行权限管理"
category: web开发
tags: [python, flask, flask-principal]
---
{% include JB/setup %}
最近做一个web版的游戏服管理工具时,在框架选型上用了之前用过一两回的[flask](http://flask.pocoo.org/),然后做到权限部分时,专门到其[官方网站](http://flask.pocoo.org/)上去找了一下相就的权限类的[扩展](http://flask.pocoo.org/extensions/),发现主要有[flask-login](https://github.com/maxcountryman/flask-login)和[flask-principal](https://github.com/ahall/flask-principal)两个权限管理类的扩展,顺手就google了一下它们的示例,于是发现了stackoverflow上的[这篇文章](http://stackoverflow.com/questions/7050137/flask-principal-tutorial-auth-authr),其中一位名为Jason Sundram的程序猿比较详细的剖析了flask-principal的实现机制(赞一个),通读之后再去翻译flask-principal的源码,发现理解起来果然容易的许多;另外,另一位名为agf的程序猿提及的[这篇文章](http://terse-words.blogspot.com/2011/06/flask-extensions-for-authorization-with.html)对flask-login和flask-principal各给出了一个示例,权衡之后,基于代码扩展性和二次开发的考虑,我选择了flask-principal做为权限管理的框架.

折腾几天后基本上搞清楚了principal的实现机制,对源码加了此[注释](https://github.com/jhezjkp/flask-principal/blob/master/flaskext/principal.py)放到了fork出来的[flask-principal项目](https://github.com/jhezjkp/flask-principal)中,并写了个[demo](https://github.com/jhezjkp/flask-principal/blob/master/demo/simpleDemo.py),有需要同学自行查阅.同时,也发现了几个小问题:

+ 只把登录用户名和认证类型存储到了会话中
+ 处理客户每次请求前都从会话中取出对应信息重新构建identity并重新与权限信息关联
+ 每次请求都读取会话信息中的权限信息并且不管有无变更都再保存一次
+ 只适合于一些授权信息基本不变更的应用,如果是一个需要灵活变更授权的应用,principal则将不适用

基于以上原因,计划对principal做一番修改,并在完成后放出.