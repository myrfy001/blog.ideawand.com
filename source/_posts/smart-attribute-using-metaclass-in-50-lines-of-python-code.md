---
title: Python之Metaclass元类详解与实战:50行代码实现【智能属性】
date: 2017-10-25 20:51:21
tags:
    - Python
---

# 0x00 背景
由于工作需要，最近学习Python元编程方面的东西。本文介绍了Metaclass的一个示例，是笔者在学习过程中编写的一个小例子，虽然只有50行代码，但其中涉及了闭包、元编程等内容，而且具有较高的实用性。 Let's Go!

> 注：本文以Python3.6为例，早期版本中元类的使用方式与本文所述方式不同。
>    上述目的也可以通过装饰器或者闭包实现，不过这里是为了学习元类，因此使用元类来做。

# 0x01 我们的目标（没有~~蛀 牙~~Bug）
为了了解元类是什么，我们先看一个很实用的例子，这样先知道了目标，再去分析后面的原理，大家思路会清晰一些。
我们要实现的目标类似于ORM框架：用一个Python类来表示一个Model，这个Model具有很多属性用来存储数据，我们可以为这些属性设置约束条件（例如数据类型等），当给这些属性进行赋值操作时，会自动根据约束检验数据是否合法，就像下面这样：



假设我们要定义一个叫做News的Model用来代表一片新闻，那么我希望能够这样：

```python
# 定义News类作为保存新闻信息的Model
# SmartAttrBase为News类的基类
# SmartAttrDesc类用来存储各个字段的约束条件
# SmartAttrBase和SmartAttrDesc类的具体实现在后文中会有介绍，目前不必关心。
class News(SmartAttrBase):
    title = SmartAttrDesc(default="", type_=str, func=lambda x: x.upper())
    content = SmartAttrDesc(default="", type_=str)
    publisher = SmartAttrDesc(default="", type_=str)
    publish_time = SmartAttrDesc(default=0, type_=int)
```

上面的News Model具有4个属性，其中3个是字符串类型，1个是整数类型，对于title字段，我们要求无论传入什么内容，都转换为大写形式进行存储。如果提供的数据类型不符，则应抛出异常，如下所示：

```python
>>> news = News()
>>> news.title
''
>>> news.title = "The quick brown fox."
>>> news.title
'THE QUICK BROWN FOX.'
>>> news.publish_time = 1508941663
>>> news.publish_time
1508941663
>>> news.publish_time = "20171025"
TypeError: Proprety [publish_time] need type of <class 'int'> but get <class 'str'>
```
<!--more-->

# 0x02 偷梁换柱

在[《深刻理解Python中的元类》](http://blog.jobbole.com/21351/)([原文](https://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python))这篇参考文献中提到，元类做的事情可以归纳为：

* 拦截类的创建
* 修改类
* 返回修改之后的类

这不就是偷梁换柱嘛。我们来看看元类偷梁换柱的例子。为了有一个直观的理解，我们仍然先不给出背后实现的代码，而是观察最终暴露出来的特性。请看下面的代码：

```python
>>> news = News()
>>> print(type(news.title))
<class 'str'>
>>> print(type(news.publish_time))
<class 'int'>
```
从之前`News`类的定义可以知道，`News`类的四个成员（title、content、publisher、publish_time）都是`SmartAttrDesc`类的实例，但是`News`类的实例中，上述四个成员的类型分别变成了对应的str和int型。

`News`类被修改了！！！

这只偷梁换柱修改`News`类的幕后黑手是什么东西？—— Metaclass ！


# 0x03 小开脑洞
来一个小热身，请问下列一行代码的含义是什么？
```Python
foo = Foo()
```
对于上述代码，大家的第一反应肯定是这样的：有一个名叫`Foo`的类，我们创建了`Foo`类的一个实例，并且将这个新创建出来的实例绑定到变量`foo`上面。一般我们会称`foo`为一个对象，`foo`可以有自己的属性，自己的方法。That's easy!

现在，让我们把脑洞开大一些。在Python中，一切事物都是对象，所以类也是一种对象。换句话说，类也可能是被实例化出来的。那么在上述代码中，`foo`是否可以代表一个类，而`Foo`是***用来生成一个类的类*** 呢？

用下面的代码来说明：
```python
# 以下代码遵循PEP8推荐的命名规范，类的名称使用大写字母开头，实例使用小写字母。
# Bar虽然是Foo的实例，但由于Bar仍然是一个类，所以Bar使用大写字母开头。
Bar = Foo()
bar = Bar()
```
请打开你的脑洞：在这里，`Foo`是一个类；`Bar`是`Foo`的实例，但是`Bar`不是一个普通的实例，因为`Bar`本身也是一个类，`Bar`这个类原本不存在，它是由`Foo`在运行过程中动态创建出来的一个类；`bar`是`Bar`的实例。
于是，我们可以说，`Foo`类是`Bar`类的类，这里的`Foo`类就是`Bar`类的***元类***， 元类就是类的类，类就是元类的实例。

# 0x04 冷静一下
看完上面拗口的描述，或多或少会有些懵。那么，先忘记元类这个东西，看点我们熟悉的，冷静一下。

看下面的代码：
```python
>>> a = 1
>>> print(type(a))
<class 'int'>
>>> print(a.__class__)
<class 'int'>
```
Python中每个实例都有一个\__class\__属性,这个属性表明了当前的这个对象是由哪个类实例化而来的。同时Python中也有一个函数`type()`可以用来检测一个实例是哪个类实例化出来的，一般而言，type(a)会返回a.\__class\__。
由于Python中任何事物都是对象，哪怕是对于最简单的数字来说都不例外，所以对于上述代码而言，变量`a`中所存储的数字`1`是`int`类的一个实例。

再看下面的代码：
```python
class Foo:
    pass
foo = Foo()
print(type(foo))
```
上述代码的输出值为：
```
<class '__main__.Foo'>
```
即表明这里的`foo`变量是`Foo`类的实例。这太正常了，不是吗？有什么好说的呢？但是，如果我们运行下面的代码，输出会是什么呢？
```python
class Foo:
    pass
print(type(Foo))
print(Foo.__class__)
```
可以看到，上述代码的运行结果为：
```
<class 'type'>
<class 'type'>
```
可以看到`Foo`是我们自己定义的一个类，并且这个类本身具有\__class\__属性，这就说明，我们定义的这个`Foo`类实际上是一个实例，它对应着一个叫做`type`的类，这个`Foo`类是`type`类的一个实例。这个`type`类是何方神圣？

回想前面元类的定义，我们定义了`Foo`类，结果发现`Foo`是一个叫做`type`的类的实例，那就说明，这个`type`类是一个元类，用`type`可以创造新的类！

# 0x05 type的历史

由于历史原因，`type`关键字在Python中有两种完全不同的含义,[Python文档](https://docs.python.org/3/library/functions.html?highlight=type#type)中对type关键字也有详细说明。
* 当type后面只有一个参数时，type是一个内置函数，用来返回一个对象所属的类
* 当type后面有3个参数时，type代表一个类，这个类在初始化时接受3个参数，并且返回一个新的type类的实例,实质上就是动态定义了一个类。



上述第一种情况，我们已经使用多次，不再赘述，现在重点看一下第二种形式。当传递给`type`三个参数时，三个参数分别是：
* name: 一个字符串，要动态创建的类的名字，这个参数会成为新创建的类的\__name\__属性
* bases: 一个tuple，要动态创建的类的基类列表，这个参数会成为新类的\__bases\__属性
* dict: 一个字典，要创建出来的类的具体内容，该字典的内容最后会成为新类的\__dict\__属性

Let's paly with `type`!

```python 
class Foo:
    pass
print(Foo)
print(type(Foo))

Bar = type("Bar", tuple(), {})
print(Bar)
print(type(Bar))
```
上述代码输出为：
```
<class '__main__.Foo'>
<class 'type'>
<class '__main__.Bar'>
<class 'type'>
```
可以看到，两种形式分别定义了两个类，这两个类几乎完全一样，单纯从输出结果根本无法判断哪个类是用class关键字定义的，哪个类是用type动态生成的。
让我们继续。
```python
class Base:
    def func_in_base(self):
        print("I am a func of Base")


class Foo(Base):
    def __init__(self, name):
        self.name = name

    def func_in_subclass(self):
        print("I am a func of subclass, my name is %s" % self.name)


def init_function_out_of_class(self, name):
    self.name = name

def normal_function_out_of_class(self):
    print("I am a func of subclass, my name is %s" % self.name)

Bar = type("Bar", (Base,), {"__init__": init_function_out_of_class,
                            "func_in_subclass": normal_function_out_of_class})

foo = Foo("Myrfy")
foo.func_in_base()
foo.func_in_subclass()
print(type(foo))
print(type(Foo))

bar = Bar("Myrfy")
bar.func_in_base()
bar.func_in_subclass()
print(type(bar))
print(type(Bar))

```
上述代码的输出为：
```
I am a func of Base
I am a func of subclass, my name is Myrfy
<class '__main__.Foo'>
<class 'type'>
I am a func of Base
I am a func of subclass, my name is Myrfy
<class '__main__.Bar'>
<class 'type'>
```
是不是很神奇？在第6~11行，我们采用传统的方法定义了一个`Foo`类；在第14~20行，又用动态创建类的方法创建了`Bar`类。`Bar`类和`Foo`类除名称不同以外，在继承和方法上的表现完全一样。
现在请注意上述输出的第8行。由于`Bar`类是使用`type`创建出来的，稍微回忆一下之前元类的概念，我们说类是元类的实例，那么在上面的例子里面，`type`是一个元类，我们实例化了一个`type`元类从而得到了一个叫做`Bar`的类，所以上述输出的第8行表明`Bar`的类型是`type`，也就是`Bar`是由`type`元类创建的。
但是……为什么第4行的输出和第8行相同？为什么我们使用class关键字定义的类也是`type`类的实例？

# 0x06 类是怎么创建出来的？
Python中的一切类都是由`type`创建出来的！！！
为了理解这个过程，我们需要看看Python解释器在逐行执行Python代码的时候究竟做了什么。以下列代码为例：
```python
class Foo(Base):
    def __init__(self, name):
        self.name = name

    def func_in_subclass(self):
        print("I am a func of subclass, my name is %s" % self.name)
```

先来看一个简化版本，更细节的操作会在下文提到。
当Python解释器遇到上述第1行后，首先看到了class关键字，于是解释器知道了我们要定义一个类，再往后看，解释器得知类的名字应该叫做Foo，再往后看，解释器发现这个类有一个父类，叫做Base。于是在第一行处理完毕后，解释器得到了两个信息：name和bases，即上文传入`type`的前两个参数。
随后，解释器会创建一个空的字典，准备存储class body的有关信息，该字典将会成为传入`type`的第三个参数。解释器继续读入上述代码的第2行，得知有一个叫做\__init\__的函数，于是解释器继续向下读入\__init\__的body的代码，创建出一个*函数对象*(function object)，然后将其插入刚才创建的字典中，取名为"\__init\__"。 同样的，解释器继续向下，创建func\_in\_subclass对应的函数对象并将其插入字典。

> 函数在Python中也是一种对象，函数对象内存储了函数的名称、所属的模块以及对应的指令字节码等信息。当Python解释器遇到def关键字时，会在内存中创建对应的函数对象，并把函数体内的代码转换为Python字节码存储在函数对象中。

至此，调用`type`来创建一个类所需的材料都已经准备好了，Python解释器会调用`type`来创建出来名为`Foo`，基类为`Base`，并且具有\__init\__和func\_in\_subclass两个方法的类。

所以，这里出现了一个令人震惊的真像：`type`类是Python中所有类的元类。平时，我们没有注意到`type`的存在，是因为Python解释器将`type`作为默认的元类。

## 关于type的一些说明
`type`是Python中很特殊的一个东西，同时也是很基础的东西：Python中用户定义的一切类最终都是通过`type`来创建的。
`type`之所以可以有如此强大的法力，是因为对`type`的调用会导致对`type`类的\__new\__方法的调用，而该方法直接对应着Python解释器[C语言代码](https://github.com/python/cpython/blob/19f68301a1295a9c30d9f28b8f1479cdcccd75aa/Objects/typeobject.c#L2326)中的一段程序，这段程序的作用是创建一个新的类型。凡是在Python语言中创建新类型的操作（也就是定义一个类），其最终都会转变为对解释器中相应代码段的调用，从而在内存中分配存储新类型的空间，创建新的结构体用来表示新创建的类型等等。
`type`在Python中会有一些特殊表现是其他任何Python类无法具备的，例如`type`类的元类是`type`本身。这是写在Python解释器代码里的一段特殊逻辑，除`type`类以外，没有任何类的元类是它自己。

用幽默一点的话来说就是，`type`类有强大的靠山，它的实现逻辑是直接写死在解释器的代码中的！所以`type`可以实现一些其他类所不能实现的东西。

# 0x07 自定义元类
## 小试牛刀
上一节中我们提到`type`是Python中默认的元类，既然有默认的，那就可以有非默认的。在Python中，如果想创建一个自定义的元类，只需要继承`type`类即可。为了使用自定义的元类，需要在定义类时指定metaclass关键字。下面是最基本的例子。
```python
# 通过继承type类来创建自己的元类
class FooMeta(type):
    pass

# 通过指定metaclass参数来指定使用哪个元类来创建当前正在定义的这个类
class Foo(metaclass=FooMeta):
    pass

print(type(FooMeta))
print(type(Foo))
```
输出为：
```
<class 'type'>
<class '__main__.FooMeta'>
```
可以看到此时`Foo`类的类是`__main__.FooMeta`而不是`type`，说明我们的`Foo`类是使用`FooMeta`元类创建出来的。需要注意的是，元类`FooMeta`本身只是一个继承了`type`的普通的类，因此`FooMeta`的元类还是`type`。总之归根到底，一切类都可以向上追溯到`type`类。任何一个类的创建，归根结底都要转换为对python解释器中有关`type`实现的代码的调用。

> 注意，Python2 不支持`metaclass=FooMeta`的写法，需要在函数体中书写`__metaclass__ = FooMeta`，详细内容请参考Python2的官方文档。

## 继续【偷梁换柱】
前文书提到元类干的“勾当”基本上来讲就是偷梁换柱，偷梁换柱是通过元类的\__new\__方法来实现的。作为元类的\__new\__方法的签名一般是这样的：
```python
class FooMeta(type):
    def __new__(cls, cls_name, cls_parents, cls_attrs):
        pass
```
上面的4个参数并不难理解。首先，对于任何类来讲，\__new\__都是一个classmethod，因此上述定义中的第一项cls与其他classmethod一样，用来表示当前类。
还记得前文中`type`在调用时需要传进去三个参数吗？这三个参数就是给`type`类的\__new\__方法使用的。因为我们自定义的元类是继承自`type`类，因此我们复写\__new\__的时候需要保留这三个参数。

有了上面的理论基础，我们开始第一次偷梁换柱！（这次是只偷不换，嘿嘿~）

```python
class FooMeta(type):
    def __new__(cls, cls_name, cls_parents, cls_attrs):
        return type.__new__(cls, cls_name, cls_parents, {})

class Foo(metaclass=FooMeta):
    def hello(self):
        print("hello")

foo = Foo()
foo.hello()
```
输出结果为
```
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'Foo' object has no attribute 'hello'
```
哈哈，我们成功偷走了`Foo`类的hello()方法！下面我们来详细解释一下背后发生的事情。
1. 解释器读取上述第1~3行代码，了解到用户想定义一个名字叫做FooMeta、父类是`type`，并且具有一个叫做\__new\__方法的类，于是Python解释器准备了一个字符串"FooMeta"、一个元组(type,)、一个函数对象\__new\__,以及一个字典{"\__new\__":\__new\__},然后使用`type`作为元类，传入上述准备好的参数，创建出来了一个名为`FooMeta`的类，这个类除了继承`type`类以外，和其他普通的类没啥区别。
2. 解释器继续读取第5~7行代码，在读取第5行时，解释器准备了一个字符串"Foo"，一个空的元组tuple()。此外，解释器还发现这里有一个metaclass关键字，于是解释器在小本本上记下来：这个类要用`FooMeta`来创建，而不是使用`type`来创建。然后解释器照常读取第6、7行，创建函数对象hello,并构建了一个字典{"hello":hello}
3. 解释器将上述准备好的材料翻译成这句话：`Foo = FooMeta("Foo", tuple(), {"hello":hello})`，然后执行它。这句话的意思很明显：创建一个`FooMeta`类的实例
4. `FooMeta`的\__new\__方法在创建新实例的过程中被调用，于是我们继续调用父类（也就是拥有强大靠山的`type`类）的\__new\__方法，唯一不同的是在这次对父类方法的调用中，代表类内部结构的字典被换成了空字典。由于Metaclass这个黑魔法是要创建一个新的类型出来，正如上文中提到的，最核心的操作还是背后的老大完成的。所以这里要向上调用`type`类的\__new\__方法。前文曾提到过，任何一个类的创建，归根结底都要转换为对`type`背后的C程序代码的调用。
5. `FooMeta`的\__new\__方法返回了一个新的类，这个类随后被赋值给一个叫做`Foo`的变量，也就是上述代码中定义的`Foo`类。只是……这个`Foo`类没有之前定义的函数了。


现在，让我们重新看一下下面这段代码。
```python
class Foo(metaclass=FooMeta):
    def hello(self):
        print("hello")
```
当你在一段Python源码中看到上面这样的类定义时，心里要知道，我看到的这3行代码只是给元类的素材而已，上面三行代码提供了类名、父类、类内部成员这三种素材，然后这些素材被交给元类去处理。元类可能返回一个和输入素材一模一样的类（使用默认的`type`作为元类），也可能返回一个被修改的面目全非的类(使用自定义元类来创建类，就上上面这样)。在我们的上一个例子中，真实获得的类是这样的：
```
class Foo(metaclass=FooMeta):      元类        class Foo():
    def hello(self):              ======>          pass
        print("hello")           
```

## 加深印象
上述演示了使用元类删除一个类方法的例子，为了大家更好的接触一下元类，我们再来试试替换和增加方法。
```python
def a_normal_function_out_of_class(self, name):
    print(name)

def another_function_out_of_class(self):
    print("A World That Has Been Modified By Metaclass")

class FooMeta(type):
    def __new__(cls, cls_name, cls_parents, cls_attrs):
        # add a new method to class
        cls_attrs["print_name"] = a_normal_function_out_of_class
        # replace a method
        cls_attrs["world"] = another_function_out_of_class

        return type.__new__(cls, cls_name, cls_parents, cls_attrs)

class Foo(metaclass=FooMeta):
    def hello(self):
        print("Hello")
    
    def world(self):
        print(" world")

foo = Foo()
foo.hello()
foo.print_name("Myrfy")
foo.world()
```
输出结果为
```
Hello
Myrfy
A World That Has Been Modified By Metaclass
```

通过修改上面的`cls_attrs`字典，我们就几乎完全掌握了一个类的创建过程。

这就是元类！

# 0x08 一人得道，惠及子孙
请看下列代码：
```
class FooMeta(type):
    pass

class Foo(metaclass=FooMeta):
    pass

class Bar(Foo):
    pass

class Baz(Bar):
    pass

print(type(Baz))
```
猜猜上面代码的输出是什么？答案是`<class '__main__.FooMeta'>`。
Python解释器会逐级向上查找各个父类，看看他们有没有指定元类，只要有前辈指定了元类，那么这个类也会使用前辈指定的元类来创建。只有这个类的各个父类都没有指定元类时，Python才会用默认的`type`作为元类。这样做带来的好处就是，不必在每个子类中显式指定元类是什么。例如你在书写一个框架，肯定不希望使用者在每次定义一个类时还要显式指定metaclass=XXXX。这时，只要定义一个类，指定好metaclass,然后让别的类继承就好了。


# 0x09 回到主题
有了上面的理论基础，现在可以开始正式编写我们的“智能属性”框架了。下面首先给出整个框架的完整代码：

```python
# Author: Myrfy (https://www.github.com/myrfy001)
# License: MIT 
class SmartAttrDesc(dict):
    pass


def get_set_wrapper(attr_name):
    # Use the closure to save the attr name
    def fget(self):
        return self._attr_values[attr_name]

    def fset(self, val):
        attr_meta = self._attr_meta[attr_name]
        type_ = attr_meta.get("type_")
        if type_ and not isinstance(val, type_):
            raise TypeError("Proprety [{}] need type of {} but get {}".format(
                attr_name, type_, type(val)))

        func = attr_meta.get("func")
        if func:
            val = func(val)
        self._attr_values[attr_name] = val

    return fget, fset


class SmartAttrMetaclass(type):
    def __new__(cls, cls_name, cls_parents, cls_attrs):

        # only type of SmartAttrDesc will became a smart attr
        _attr_meta = {name: val for name, val in cls_attrs.items()
                      if isinstance(val, SmartAttrDesc)}
        cls_attrs["_attr_meta"] = _attr_meta

        # Override propreties
        for attr_name in _attr_meta:
            cls_attrs[attr_name] = property(*get_set_wrapper(attr_name))

        new_cls = type.__new__(cls, cls_name, cls_parents, cls_attrs)
        return new_cls


class SmartAttrBase(metaclass=SmartAttrMetaclass):
    def __new__(cls):
        new_inst = object.__new__(cls)
        new_inst._attr_values = {}
        for attr_name, attr_meta in cls._attr_meta.items():
            setattr(new_inst, attr_name,  attr_meta.get("default"))
        return new_inst
```


## “智能属性”描述类
代码的第3~4行实现了用来描述一个“智能属性”特征的`SmartAttrDesc`类。从文章开头的示例代码和名称不难看出这个类的作用是描述一个属性，该类在初始化时接受不定参数用来描述该属性的特征，并将特征保存起来。可以看出其作用与初始化形式均与python中的dict类非常相似，于是采用拿来主义：
```python
class SmartAttrDesc(dict):
    pass
```
这里不能直接用dict吗？其实也可以，不过呢，重新起一个名字可以具有更好的可读性。此外，后文中的元类会判断一个类成员是不是`SmartAttrDesc`的实例，只有类成员是`SmartAttrDesc`时，元类才会将其替换成对应的属性类型，如果直接使用dict，则可能导致元类修改了他不该修改的成员。

## SmartAttrMetaclass概述
代码的第27~40行定义了`SmartAttrMetaclass`元类。
在31~32行，我们创建了一个名为`_attr_meta`的字典，用来保存待创建的类中所有“智能属性”的描述信息。在这里使用了isinstance函数来筛选出所有的`SmartAttrDesc`实例。
在第33行，我们将刚才生成的`_attr_meta`字典放入到将要创建的类中，这样，由该元类创建的类将包含一个`_attr_meta`属性，内部存储了原来定义的所有“智能属性”的描述信息。
第36~37行，原来类中定义的`SmartAttrMetaclass`类型的成员被替换为由property函数生成的属性成员，最终创建的类中将不包含`SmartAttrMetaclass`类型的成员。

## fget和fset函数的实现
代码第9~22定义了传递给`property`函数的`fget`和`fset`函数，在`fget`函数比较简单，只是返回属性的值，而`fset`函数则根据之前创建的`_attr_meta`类属性中保存的规则来进行校验，若尝试赋给属性的值不合法则会抛出异常。
代码中出现的`_attr_values`属性是在类实例化时设置的一个字典，用来保存各个属性的属性值，下文再详细介绍。

需要注意的是，我们书写的`fget`和`fset`函数需要应对不同的属性，因此我们需要知道是对哪个属性的访问操作触发了`fget`或者`fset`。很不幸的是，`property`函数所接收的`fget`和`fset`函数的签名并不包含当前被调用的属性的名字。所以我们需要想办法把属性的名字与`fget`和`fset`函数绑定到一起。这个工作可以通过`functools.partial`来实现，也可以通过自己通过闭包来实现，这里选择第二种方式，可以看到代码的第9~22是写在第7行定义的`get_set_wrapper`函数中。

## SmartAttrBase类的实现
第43~49行定义了`SmartAttrBase`类，可以明显看到，该类指定了元类，因此所有继承自该类的子类都会使用`SmartAttrMetaclass`作为元类。
在这里，我们重写了`SmartAttrBase`类的`__init__`方法，其目的是为新创建出来的实例创建`_attr_values`成员（一个字典），并将各个属性的默认值存入其中。`fget`和`fset`也会读写其中的值。



# 0x0A 结语

以上是笔者学习Python Metaclass的过程中所了解到的一些原理性质的东西，并通过一个不到50行但很实用的小例子展示了Python中元类的用法。由于笔者水平有限，如有错误，请到[https://github.com/myrfy001/blog.ideawand.com](https://github.com/myrfy001/blog.ideawand.com)提出Issue, 我会尽快改正。

# 参考文献
* [《深刻理解Python中的元类》](http://blog.jobbole.com/21351/)([原文](https://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python))
* [Understanding Python metaclasses](https://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses/)
* [Under the hood of Python class definitions](https://eli.thegreenplace.net/2012/06/15/under-the-hood-of-python-class-definitions)
* [The built-in keyword type means a function or a class in python?](https://stackoverflow.com/questions/47006895/the-built-in-keyword-type-means-a-function-or-a-class-in-python)
* [Python文档关于type的描述](https://docs.python.org/3/library/functions.html?highlight=type#type)