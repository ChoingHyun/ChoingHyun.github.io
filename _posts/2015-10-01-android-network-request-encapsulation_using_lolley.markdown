---
layout:     post
title:      "Android开发之基于Volley请求的封装"
subtitle:   "Volley请求的二次封装"
date:       2015-10-01
author:     "Eric"
header-img: "img/post-bg-android.jpg"
tags:
    - 编程
    - Android
    - 网络
    
---
**什么是Volley**
废话连篇，在稍微介绍下，Volley是Google在2013年I/O大会推出的，相对来说比较成熟，适用于数据量较小，请求频繁地情况，不擅长于大文件的请求响应。现在也有其他非常优秀的第三方网络请求库，OKHTTP，有时间待做研究。
<br>

**为什么要对Volley进行封装**
- 为了代码的重用。在工程中，如果有较多API的请求，将出现大量重复的Volley库请求、回调代码，这明显是不能忍受的。
- 为了做到请求、响应、数据的解析之间的分离解耦。

**整体思路**
在设计过程中，主要提供了一下几个功能相关的类：
1. RequestManager 是请求调用的第一层级，在发送请求时，首先调用该类对象的不同请求方法，方法中对请求参数进行打包（放入Map对象），然后调用RequestHelper的请求方法，进行网络请求
2. RequestHelper Volley请求在这里发出，同时网络请求策略（超时、重复请求）也将在这里进行配置，得到的响应传给ResponseHandler进行分发
3. ResponseHandler 进行结果的解析，以及回调
4. Activity注册回调，并在网络回调中进行处理