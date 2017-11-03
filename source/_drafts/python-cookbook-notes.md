---
title: python_cookbook_notes
date: 2017-10-10 13:22:12
tags:
---

*语法放在赋值语句左侧用于接受多个变量， python3中的新语法？


slice函数创建切片对象
http://python3-cookbook.readthedocs.io/zh_CN/latest/c01/p11_naming_slice.html
字符串切片不仅可以指定start和stop, 还可以指定step


itemgetter, attrgetter的执行效率要比lambda表达式略快一些(Why?)
itemgetter和attrgetter可以被pickle, lambda不行  https://stackoverflow.com/questions/11287207/why-should-i-use-operator-itemgetterx-instead-of-x


(x for x in range(10)) 返回生成器  [x for x in range(10)]  返回列表

ChainMap

string.startswith和endswith可以传入一个tuple，用来检测多种可能。

fnmatch