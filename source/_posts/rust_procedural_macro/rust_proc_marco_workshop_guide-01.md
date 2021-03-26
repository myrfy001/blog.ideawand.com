---
title: Rust过程宏系列教程(1)--搭建过程宏开发环境并熟悉基本原理
date: 2021-02-27 22:09:26
tags: Rust,过程宏,proc-macro
---

如果想用Rust去开发大型项目，特别是框架级别的项目，那么Rust的过程宏(proc-macro)机制肯定是一个必须掌握的技能。极客幼稚园公众号从本期开始，将进行一个系列连载，以文字 + 视频的形式，介绍过程宏的相关知识点，并通过几个项目实战，手把手书写几种不同类型的过程宏。我的视频将会上传至B站，文字版本与视频版本内容会大致相符，但文字版本对原理的讲解会更加清晰，大家看到的是一个最终【正确】的版本，而视频版本对于编写和调试过程的展示更加友好，大家可以看到编写一个过程宏的整个过程中是如何一步步阅读文档、踩坑的。另外，这一系列文章也是我本人近期在学习过程宏的过程中边学边写的，可能有很多野路子，如果大家有更好的实现方式或者编码方式，欢迎与我交流讨论。

视频教程在这里，完整文字版本请点击视频下方的【阅读原文】
<iframe src="//player.bilibili.com/player.html?bvid=BV1Ny4y1b7iS&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>


<!--more-->

过程宏的用处、背景信息等，我就不再介绍了，看这系列教程的同学应该都是有明确目的才来看的，我们就直接上手实战，请大家准备好电脑，我们从如何搭建一个Rust过程宏的开发调试环境入手。

新建一个文件夹，我们要在其中建立两个**嵌套**的crate。因为在rust中，有以下几个约束：
* 过程宏必须定义在一个独立的crate中。不能在一个crate中既定义过程宏，又使用过程宏。
  * 原理：考虑`过程宏`是在编译一个crate之前，对crate的代码进行加工的一段程序，这段程序也是需要编译后执行的。如果定义过程宏和使用过程宏的代码写在一个crate中，那就陷入了死锁:
    * 要编译的代码首先需要运行过程宏来展开，否则代码是不完整的，没法编译crate。
    * 不能编译crate，crate中的过程宏代码就没法执行，就不能展开被过程宏装饰的代码
* 过程宏必须定义定义在`lib`目标中，不能定义在`bin`目标中

创建实验环境目录结构的Shell命令如下：
```shell
cd blog.ideawand.com # 进入我们的工作目录
mkdir rust_proc_macro_guide && cd rust_proc_macro_guide
cargo init --bin  # 创建使用过程宏的crate

mkdir proc_macro_define_crate && cd proc_macro_define_crate
cargo init --lib # 创建定义过程宏的crate
```

我们在上面创建了一个嵌套的层级结构，其实这就是为了便于管理。如果你把`rust_proc_macro_guide`文件夹和`proc_macro_define_crate`文件夹平级放置，或者把他俩放到不同的磁盘分区上去，都是没问题的。以上面的操作为例，创建完成以后，你的目录结构应该是这样的：
```
blog.ideawand.com
└── rust_proc_macro_guide
    ├── Cargo.lock
    ├── Cargo.toml
    ├── proc_macro_define_crate  # 这是一个用于定义过程宏的独立的crate
    │   ├── Cargo.toml
    │   └── src
    │       └── lib.rs  # 我们的过程宏代码在这里编写
    └── src
        └── main.rs  # 在这里使用我们定义的过程宏
```

> 重点：后文中为了表述方便，我们将定义过程宏的crate称为内层crate，将使用过程宏的crate叫外层crate


接下来，我们修改两个项目的`Cargo.toml`，配置我们的实验环境。

首先在内层`proc_macro_define_crate/Cargo.toml`中添加`[lib]`节点并在下面增加`proc-macro = true`，表示这个crate是一个`proc-macro`，增加这个配置以后，这个crate的特性就会发生一些变化，例如，这个crate将只能对外导出内部定义的过程宏，而不能导出内部定义的其他内容。
```toml
[lib]
proc-macro = true
```

然后在`dependencies`节点下，添加如下两个依赖包，这也是开发rust过程宏必备的两个依赖包，后面再详细说。其中syn包的`extra-traits`特性是为了方便我们后续打印调试信息。
```toml
[dependencies]
quote = "1"
syn = {features=["full","extra-traits"]}
```

然后打开外层crate的`rust_proc_macro_guide/Cargo.toml`文件，添加上面的内层crate作为依赖。我们已经说过了，这两个crate本质上是独立的，这里仅仅是通过依赖关系把它们给关联起来了。只要下面这两行配置文件中的路径写的对，这两个包在磁盘上存在哪个目录下都行。

```toml
[dependencies]
proc_macro_define_crate = {path="./proc_macro_define_crate"}
```


接下来，我们开始写一个最简单的过程宏。编辑`proc_macro_define_crate/src/lib.rs`,代码如下，对照代码下面的说明食用口味更佳：
```rust
use proc_macro::TokenStream;

#[proc_macro_attribute]
pub fn mytest_proc_macro(attr: TokenStream, item: TokenStream) -> TokenStream {
    eprintln!("{:#?}", attr);
    eprintln!("{:#?}", item);
    item
}
```
* 我接下来会提到`编译原理`这几个字，不过别慌，没学过也没关系
* 我们引入了一个叫做`TokenStream`的类型，稍微了解一点点编译原理的同学看到`Token`这个词应该就会想到，这个八成和源代码做词法分析有关。实际上，这个类型表示的就是输入的源码文件，经过词法分析后的结果。我们在后面会给大家演示里面保存的数据是什么样子的。保证给大家一个清晰直观的认识。
* `#[proc_macro_attribute]`是在告诉编译器我们在定义一个"属性式"的过程宏
  * 它还有两个兄弟：`#[proc_macro]`和`#[proc_macro_derive]`,分别用于定义"函数式"和"派生式"两种类型的过程宏
* 接下来的函数名称，就是我们的过程宏的名称
* 函数输入有两个参数，分别是`attr`和`item`，别紧张，后面看例子就清楚了
* 我们这里在打印的时候使用了`eprintln!`，这是因为cargo在调用rustc进行编译的时候，stdout的输出会被cargo吞掉，而stderr上的输出会在控制台被打印出来，所以我们要把输出打印到stderr上。
* Rust过程宏的本质就是一个编译环节的过滤器，或者说是一个中间件，它接收一段用户编写的源代码，做一通酷炫的操作，然后返回给编译器一段经过修改的代码。
  * 函数的返回值直接返回了`item`，表示的含义是，我什么都不修改，输入代码是什么样，还原封不动给回去。
    * 而我们后面要学习的，就是如何编写更复杂的逻辑，返回一个加工后的`item`给编译器用

> 入坑提示：
> * rust的过程宏代码目前没有比较好的调试方法，print大法应该是最好用的。
> * 做个广告，本篇文章出自微信公众号【极客幼稚园】，欢迎关注，您也可以访问我的个人博客http://blog.ideawand.com
> * 如需转载，请注明出处


这段过程宏定义的代码是写完了，但是现在还不能直接用，我们没法直接运行它，因为它是由编译器在编译其他代码的过程中调用的，所以我们现在回到外层crate中，使用我们编写的过程宏，这样，在编译外层crate的时候，我们写的过程宏代码就会被执行啦。

我们打开`rust_proc_macro_guide/src/main.rs`，输入如下代码,注意其中的blog、ideawand、com等大家可以所以换成其他的标识符：

```rust
use proc_macro_define_crate::mytest_proc_macro;

#[mytest_proc_macro(blog(::ideawand::com))]
fn foo(a:i32){
    println!("hello, blog.ideawand.com, hello, 极客幼稚园");
}
```

保存之后呢，我们在`rust_proc_macro_guide`目录下，运行`cargo check`命令，我们在这里不需要执行build，check的过程就会将宏进行展开。也可以看到我们的`main.rs`里面并没有定义`main`函数，因为我们要关心的是过程宏展开的过程，还没到具体编译那一步呢，没有`main`函数，`cargo check`虽然会在最后报错，但是不影响我们观察过程宏展开情况。（顺便，通过我们在过程宏中打印的输出信息和编译报错信息打印在控制台的先后顺序，我们可以从侧面证明，宏展开是在编译之前完成的）。

`cargo check`的输出结果如下所示，比较长，我先给出食用建议，然后大家再开始食用，以免无法品鉴其中的美味。
* 一定要对照着用户输入的源代码来阅读`attr`和`item`这两个`TokenStream`类型数据的内容，我们在过程宏的定义中先后打印了`attr`变量和`item`变量的值，为了便于大家阅读，我把这两个输出在下面分别给大家展示。
* 仔细观察用户源码中每一个元素在`TokenStream`类型中的表示形式


下面这一行用户原始代码，被编译器解析为`TokenStream`后，作为`attr`变量，传递给我们的过程宏。解析后的`attr`内容也列在下面供对照阅读：
```rust
#[mytest_proc_macro(blog(::ideawand::com))]
```
```rust
TokenStream [
    Ident {
        ident: "blog",
        span: #0 bytes(69..73),
    },
    Group {
        delimiter: Parenthesis,
        stream: TokenStream [
            Punct {
                ch: ':',
                spacing: Joint,
                span: #0 bytes(74..76),
            },
            Punct {
                ch: ':',
                spacing: Alone,
                span: #0 bytes(74..76),
            },
            Ident {
                ident: "ideawand",
                span: #0 bytes(76..84),
            },
            Punct {
                ch: ':',
                spacing: Joint,
                span: #0 bytes(84..86),
            },
            Punct {
                ch: ':',
                spacing: Alone,
                span: #0 bytes(84..86),
            },
            Ident {
                ident: "com",
                span: #0 bytes(86..89),
            },
        ],
        span: #0 bytes(73..90),
    },
]
```
我们先分析`attr`变量中的内容，这样后面再看`item`变量的内容就小菜一碟了。我们可以看到：
* `TokenStream`以树形结构的数据组织，表达了用户源代码中各个语言元素的类型以及相互之间的关系
* 每个语言元素都有一个`span`属性，记录了这个元素在用户源代码中的位置。
* 不同类型的节点，有各自独有的属性，其他我们在这个例子中没有涉及到的节点类型，大家可以去阅读文档
* `Ident`类型表示的是一个标识符，这可能是我们后面会用的非常频繁的一个类型。变量名、函数名等等，都是标识符。
* `TokenStream`里面的信息，是没有语义信息的，比如在上面的例子中，路径表达式中的双冒号`::`被拆分为两个独立的冒号对待，`TokenStream`并没有把它们识别为路径表达式，同样，它也不区分这个冒号是出现在一个引用路径中，还是用来表示数据类型。
* 针对`attr`属性而言，其中不包括宏自己的名称的标识符，它包含的仅仅是传递给这个过程宏的参数的信息

看完`attr`,我们再来看看`item`参数的值。`item`参数对应的是被过程宏修饰的代码块，也就是以下三行代码：
```rust
fn foo(a:i32){
    println!("hello, blog.ideawand.com, hello, 极客幼稚园");
}
```
对应的`TokenStream`为：
```rust
TokenStream [
    Ident {
        ident: "fn",
        span: #0 bytes(93..95),
    },
    Ident {
        ident: "foo",
        span: #0 bytes(96..99),
    },
    Group {
        delimiter: Parenthesis,
        stream: TokenStream [
            Ident {
                ident: "a",
                span: #0 bytes(100..101),
            },
            Punct {
                ch: ':',
                spacing: Alone,
                span: #0 bytes(101..102),
            },
            Ident {
                ident: "i32",
                span: #0 bytes(102..105),
            },
        ],
        span: #0 bytes(99..106),
    },
    Group {
        delimiter: Brace,
        stream: TokenStream [
            Ident {
                ident: "println",
                span: #0 bytes(112..119),
            },
            Punct {
                ch: '!',
                spacing: Alone,
                span: #0 bytes(119..120),
            },
            Group {
                delimiter: Parenthesis,
                stream: TokenStream [
                    Literal {
                        kind: Str,
                        symbol: "hello, blog.ideawand.com, hello, 极客幼稚园",
                        suffix: None,
                        span: #0 bytes(121..171),
                    },
                ],
                span: #0 bytes(120..172),
            },
            Punct {
                ch: ';',
                spacing: Alone,
                span: #0 bytes(172..173),
            },
        ],
        span: #0 bytes(106..175),
    },
]
```

上面的结果我就不再深入分析了，相信通过上面的例子，大家会发现，我们的`代码`也被转换成了一种`数据`,也就是一种`代码即数据`的思想，这可能也和Rust借鉴了Lisp语言有关吧，都说Rust上能看到很多种语言的影子。`TokenStream`类型的数据，可以和Rust源代码之间做等价的转换。

有了上面的铺垫，我们现在就很容易理解了：所谓的Rust过程宏，就是我们可以自己修改上面的`item`变量中的值，从而等价于`加工原始输入代码`,最后将加工后的代码返回给编译器即可。

在这里，我们再补充一个实验，来证明`TokenStream`只是一系列符号的组合，和语义无关。我们将上面的实例修改为如下的形式,虽然我们写了一堆没有任何含义的乱码，可是编译器依然会按照词法规则，将其进行切分，然后将其变为一个与之等价的`TokenStream`：
```rust
#[mytest_proc_macro(aa@@3 324 +_ {} %^*$^%# [] [  &} )]
fn foo(a:i32){
    println!("hello, blog.ideawand.com, hello, 极客幼稚园");
}
```

通过上面的实验，我们就会认识到，`TokenStream`还是一种比较低级的表达形式，手工去写这样的树形结构，也一定会让人发疯，因此，我们就要借助`syn`和`quote`这两个Rust库来提供更友好的开发体验。他们可以把`TokenStream`转化为具有语义信息，抽象程度更高的一种数据结构——`语法树`。我们把之前的过程宏定义代码修改成下面这个样子，大家可以先试着自己理解一下，然后再看后面的要点说明：
> 注意：我们这里提到的是`语法树(Syntax Tree，ST）`，大家可能经常听到的另一个词是`抽象语法树（Abstract Syntax Tree，AST）`。通常，抽象语法树是指经过引用消解等一系列后续操作而得到的，具有高度抽象表达能力的数据结构，而Rust的过程宏系统并没有引用消解等能力，因此我们将下面过程得到的数据结构成为AST是不严谨的。但是，由于AST被大家说的太多了，我在后续的介绍中也可能出现笔误或者视频中口误，大家根据上下文，理解我想表达的意思即可。
```rust
use proc_macro::TokenStream;
use syn::{parse_macro_input, AttributeArgs, Item};
use quote::quote;

#[proc_macro_attribute]
pub fn mytest_proc_macro(attr: TokenStream, item: TokenStream) -> TokenStream {
    eprintln!("{:#?}", parse_macro_input!(attr as AttributeArgs));
    let body_ast = parse_macro_input!(item as Item);
    eprintln!("{:#?}", body_ast);
    quote!(#body_ast).into()
}
```
代码要点：
* 与之前相比，我们新引入了`syn`包的`AttributeArgs`、`Item`两个数据类型和`parse_macro_input!`宏，以及`quote`包中的`quote!`宏。
* `parse_macro_input!`宏将`TokenStream`类型的参数解析为语法树
  * 注意，`parse_macro_input!`宏内部用到了`as`关键字，这里的`as`并不是rust内置的用来做类型转换的关键字，而是`parse_macro_input!`宏提供的一种用来做类型注解的方法，通过这样的一种写法，告诉`parse_macro_input!`宏应该如何解析`TokenStream`，或者说，告诉了`parse_macro_input!`宏这些`TokenStream`来自于代码的那些环节。
    * 这里又体现出了Rust宏的强大之处，通过宏，这里让人似乎感觉到我们扩展了Rust的语法，或者说，通过宏实现了一种领域特定语言(DSL)
    * 虽然`attr`和`item`都是`TokenStream`类型的数据，但是不同的类型提示，会导致`parse_macro_input!`宏输出不同类型的结果
      * `parse_macro_input!(attr as AttributeArgs)`输出的是`Vec<NestedMeta>`类型的语法树节点
      * `parse_macro_input!(item as Item)`输出的是`Item`类型的语法树节点
* 通过`quote!`宏，可以将语法树节点及其子节点重新转换为`TokenStream`
  * 这里的`#body_ast`的写法，同样是通过`quote!`宏实现的一种自定义语法，并不是合法的Rust语言表达式。
  * `quote!`宏返回的结果实际是`proc_macro2::TokenStream`类型的数据，所以要通过`into()`进行一下转换，变成`proc_macro::TokenStream`类型的数据，这是有一定的历史背景的
    * 由于Rust语言还处在年幼阶段，编译器内部结构变化也会比较大，所以rust编译器给我们提供的接口是没有语义的，近似于一堆字符串片段的`proc_macro::TokenStream`类型的数据。这个`proc_macro`包是由rust官方提供的。
    * `quote`、`syn`、`proc_macro2`这些包，都是由社区或者第三方提供的，这些包基于官方的`proc_macro::TokenStream`做了很多额外的功能，目的是为了方便我们更方便地去处理、编辑、生成`proc_macro::TokenStream`
    * 从这一点来体会，rust提供了标准（`proc_macro::TokenStream`），通过这些标准，可以在上层发展各种生态（`quote`、`syn`、`proc_macro2`）来扩展rust。

我们回到外层crate所在的目录，再次执行`cargo check`观察过程宏展开的结果，这次我们可以看到`TokenStream`解析得到的语法树结构如下：

```rust
#[mytest_proc_macro(blog(::ideawand::com))]
```
对应的语法树为：
```rust
[
    Meta(
        List(
            MetaList {
                path: Path {
                    leading_colon: None,
                    segments: [
                        PathSegment {
                            ident: Ident {
                                ident: "blog",
                                span: #0 bytes(69..73),
                            },
                            arguments: None,
                        },
                    ],
                },
                paren_token: Paren,
                nested: [
                    Meta(
                        Path(
                            Path {
                                leading_colon: Some(
                                    Colon2,
                                ),
                                segments: [
                                    PathSegment {
                                        ident: Ident {
                                            ident: "ideawand",
                                            span: #0 bytes(76..84),
                                        },
                                        arguments: None,
                                    },
                                    Colon2,
                                    PathSegment {
                                        ident: Ident {
                                            ident: "com",
                                            span: #0 bytes(86..89),
                                        },
                                        arguments: None,
                                    },
                                ],
                            },
                        ),
                    ),
                ],
            },
        ),
    ),
]
```
可以看到，在上面的语法树中，已经包含了通过上下文推导出的语义信息，`::ideawand::com`已经不再被认为是4个冒号和两个标识符了，在语法树中，这个结构被一个`Path`类型的节点所表示，而`Path`类型中，又有`segments`字段来表示其中的`ideawand`和`com`。通过这个例子，大家应该可以直观感受到，语法树是经过语义分析得到的结果，是一种对源代码更加完善的表示，每一个元素不再是冰冷的字符，而是拥有了特定的含义。同样也正因为如此，传入`parse_macro_input!`的TokenStream必须是符合Rust语法规则的Token序列才行，前文中我们用来做实验的`#[mytest_proc_macro(aa@@3 324 +_ {} %^*$^%# [] [  &} )]`虽然可以用`TokenStream`来表示，但是却不能被解析为语法树节点，你可以试一下，会得到什么样的报错信息。

语法树节点同样有很多种类型，上面的示例中只涉及到很少的几种，大家可以在读完本篇文章以后，自行去阅读相关文档。

本篇文章到这里先告一段落，总结一下，本篇主要是为了给大家介绍Rust的过程宏系统的实现机制，并且带领大家搭建了一个实验工程的目录结构，在其中直观感受了一下在过程宏实现的各个环节中，我们面对的数据是什么样子的。

本篇文章所实现的过程宏，并没有对`TokenStream`做任何修改，只是把代码原封不动还给了编译器。因为如何修改、生成代码又是一个大话题，需要结合项目动手实战才能有收获，我会在接下来的文章中带领大家实战几个过程宏的编写。