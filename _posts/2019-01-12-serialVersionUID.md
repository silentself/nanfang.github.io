---
layout: post
title:  聊聊序列化和反序列化
categories: [java]
tags: [serivalVersionUID]
comments: true
---

由于本人一向习惯实现序列化之后不生成序列化id直接去警告，故而与项目中其他协作者出现不一致的行为，可是也从来没有出现过错误，没有出现并不代表没有出现的可能，所有详细了解序列化id之后记录下来两种编码的方式异同

<!---more--->

查过一些博客、Serializable类以及