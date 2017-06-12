---
title: Unicode中的【合字】现象以及Python中的相关Bug
date: 2017-06-11 22:46:46
tags:
---

* 注意：本篇博客涉及字符编码及其显示问题，不同平台、浏览器、字体的显示效果可能不同。如无法正常显示，请尝试更换字体。

最近在调试公司的一个服务，在使用正则表达式对抓取的网页进行分词处理时，采用了  `\w+`  作为分词规则。在印地语（Hindi，属于梵文）作为输入时出现了一个Badcase。

输入内容为：

```python
u"बसपा, भाजपा में सांठगांठ, मुसलमान सपा के पक्ष में एकजुट: आजम खान”
```

经过匹配后重组的句子变为：

```python
u"बसप भ जप म स ठग ठ म सलम न सप क पक ष म एकज ट आजम ख न”
```
对比前后两段文字，可以看到经过正则匹配的文字丢失掉了一些细节， 比如某些符号上面的小黑点等。

造成这个现象的原因是Unicode里面存在的 “合字” 机制（刷新了我对字符的理解），我们以字符`में`为例进行说明。
	
在IPython下查看这个符号的组成：

```python
	In [152]: a = u'में'

	In [153]: a
	Out[153]: u'\u092e\u0947\u0902'	
```

可以看到，我们看起来的一个“字符”实际上是由3个Unicode字符组成的，那么，我们分别看一下这三个Unicode字符分别是什么：

```python
	In [154]: print a[0]
	म

	In [155]: print a[1]
	 े

	In [156]: print a[2]
	 ं
```

后两个字符，下面是一个虚线画出的圆圈占位符，这种符号可以和前面的字母组合为新的显示形式，组合后，原来的虚线圆环就不会再显示了。

这种现象，个人感觉非常类似用汉字中的偏旁部首来描述一个汉字的感觉。
接下来，按照这个思路，我们可以自己造一个符号出来 ;)

下面的这个示例，输出应该为1个字符A，并且字符A的上面有一个 从左上至右下的斜杠以及一个点 组成的上标。如果浏览器显示为两个字符或者三个字符，则说明显示问题。复制该段代码到Python中执行，在我的OSX操作系统上可以看到正确的输出。

```python
	In [260]: b = u'\u0041\u0947\u0902'

	In [261]: print b
	Aें
```

下面，问题来了，看看Python的自带正则表达式模块re是怎么处理这种字符的

```python
	In [157]: re.findall(r'\w+', a, re.UNICODE)
	Out[157]: [u'\u092e’]
```

可以看到，re只把字根保留了下来，而其他部分被当做标点符号给去除掉了。

解决方案是，使用第三方正则库regex

```python
	In [160]: regex.findall(r'\w+', a, regex.UNICODE)
	Out[160]: [u'\u092e\u0947\u0902']
```

	
最后一句话：使用Python的正则表达式库处理不常见语言要小心。


<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Creative Commons Attribution-ShareAlike 4.0 International License</a>.