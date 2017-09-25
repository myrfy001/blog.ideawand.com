---
title: Elasticsearch的mapping设置：enabled, index, doc_values, store, _source到底是什么鬼？
date: 2017-09-23 16:32:28
tags: Elasticsearch,存储优化
---

## 0x00 背景介绍
最近尝试用ES + Kibana来快速搭建一个全新的可视化平台，有机会仔细阅读了一下ES的文档，发现mapping里有很多设置选项，初次看时令人眼花缭乱，若设置不当，有可能浪费存储空间，也有可能导致无法使用Aggregations，故在此记录一下重点内容。如有错误，恳请[点击这里提issue](https://github.com/myrfy001/blog.ideawand.com),我会及时改正。

本文参照的版本为 Elasticsearch 5.6

<!--more-->

## 0x01 配置项速查
这里列出了各个选项的名称、作用以及注意事项，仅供速查使用。详细解释请阅读下文。

| 配置项 | 作用 | 注意事项 | 默认值 |
|-------|-----|---------|-----|
|index|是否加入倒排索引|<ul><li>关闭后无法对其进行搜索</li><li> 字段仍会存储到_source和doc_values</li><li> 字段可以被排序和聚合</li></ul> |开启|
|_source|存储post到ES的原始文档|<ul><li>会占用很多存储空间。</li><li>数据压缩存储，读取会有额外解压开销</li><li>不需要读取原始字段内容可以考虑关闭</li><li>关闭后无法reindex</li></ul>|开启|
|_all|提供跨字段全文检索|<ul><li>会占用额外存储空间</li><li>没有跨字段全文检索需求可以关闭</li></ul>|开启|
|store|是否单独存储该field|<ul><li>会占用额外存储空间</li><li>与_source独立，同时开启store和_source则会将该字段原始内容保存两份</li><li>字段单独存储，数据在磁盘上不连续，若读取多个字段需要seek多次</li><li>如需读取多个字段，需权衡比较_source与store效率</li></ul>|关闭|
|doc_values|排序聚合支持|<ul><li>会占用额外存储空间</li><li>与_source独立，同时开启doc_values和_source则会将该字段原始内容保存两份</li><li>数据在磁盘上采用列式存储</li><li>关闭后无法使用排序和聚合</li></ul>|开启|
|<s>fielddata</s>|doc_values的前身|已废弃||
|term_vector|记录分词信息|<ul><li>会占用额外存储空间</li><li>highlight功能可以借助其加速</li></ul>|关闭|
|enabled|是否对该字段进行处理|<ul><li>关闭后，只在_source中存储</li><li>类似index与doc_value的总开关</li></ul>|开启|

## 0x02 冗余的存储--概述
众所周知，ES是以其全文检索功能而著名的，提到全文检索，大家的反应就是倒排索引；事实上，**一个输入的文档会被ES以多种方式进行存储**，从而使得ES在全文检索外还支持排序、聚合等功能。

ES的检索功能在底层使用的是Lucene，根据[[1]](https://www.elastic.co/blog/elasticsearch-as-a-column-store)描述，Lucene的索引包含以下部分：

>A Lucene index is made of several components: an inverted index, a bkd tree, a column store (doc values), a document store (stored fields) and term vectors.

其中：
* inverted index: 倒排索引，不用多说。
* bkd tree: Block k-d tree，用于在高维空间内做索引，如地理坐标的索引。
* column store: 列式存储，可以充分利用操作系统的缓存，批量读取连续的数据以提高排序和聚合的效率。（说道列式存储，联想到了Cassandra）
* document store: 用于存储文档，功能和_source字段有重合，下文详细介绍区别。
* term vectors: 用于存储各个词在文档中出现的位置等信息。

在很多场合下，我们并不需要上述全部信息，因此可以通过设置mappings里面的属性来控制哪些是我们需要存储的，哪些不需要。

## 0x03 冗余的存储--深入
在ES的mapping设置里，有如下parameters和eta-Fields可以设置，每一项都对应着一种存储形式：
* index
* _source
* _all
* store
* doc_values
* fielddata
* term_vector
* enabled



上述列表中的项目是在配置mappings时经常遇到的，其中有部分选项看似功能相似，实则不同，我们慢慢来分析：

#### `index`
这个属性用于控制一个字段是否需要被索引，默认情况下是开启的。如果关闭了index，则该字段的内容不会被analyze, 也不会存入倒排索引，即意味着该字段无法被搜索。

#### `_source`
这个字段的作用是存储post到API接口的原始json文档。为什么要存储原始文档呢？因为ES采用倒排索引对文本进行搜索，而倒排索引无法存储原始输入文本。简单来说，一段文本交给ES后，首先会被analyzer打散成单词，为了保证搜索的准确性，在打散的过程中，会去除文本中的标点符号，统一文本的大小写，甚至对于英文等主流语言，会把发生形式变化的单词恢复成原型或词根，然后再根据统一规整之后的单词建立倒排索引，经过如此一番处理，原文已经面目全非，因此需要有一个地方来存储原始的信息，以便在所搜到这篇文档的时候能够把原文返回给查询者。那么：
* 一定要存储原始文档吗？ 不一定！ 如果只关心一篇文档是否存在，而不关心这个文档的内容，可以选择不保存_source字段。
* 可以只保存原始文档的一部分到_source里面吗？ 可以，ES提供了过滤规则，可以只将一部分字段存入_source中。

如果不存储_source或仅存储部分内容，可以大量减小ES的存储占用量。但是，这样做有负面影响吗？ 有！
* 不能获取到原文（废话）
* 无法reindex：如果存储了_source，当index发生损坏，或需要改变mapping结构时，由于存在原始数据，ES可以通过原始数据自动重建index，如果不存_source则无法实现
* 无法在查询中使用script：因为script需要访问_source中的字段。


#### `_all`
这个字段的作用是提供跨字段查询的支持。ES在查询的过程中，需要指定在哪一个field里面查询。例如下面的文档
```json
{
    "name": "smith",
    "email": "John@example.com"
}
```
用户在查询时，想查询叫做John的人，但是并不知道这个John出现在`name`字段中还是出现在`email`字段中，由于ES是为每一个字段单独建立索引，所以用户需要以John为关键词发起两次查询，分别查询`name`字段和`email`字段。
如果开启了_all字段，则ES会在索引过程中创建一个虚拟的字段_all，其值为文档中各个字段拼接起来所组成的一个很长的字符串，例如上面的例子，_all字段的内容为字符串"smith John@example.com"。随后，该字段将被分词打散，与其他字段一样被收入倒排索引中。由于该字段的内容都来自_source字段，因此默认情况下，该字段的内容并不会被保存，可以通过设置store属性来强制保存_all字段。
由于_all字段包含了所有字段的信息，因此可以实现跨字段的查询，即用户不用关心要查询的关键词在哪个字段中，ES可以将包含该关键词的文档全部检索出来，又用户进行下一步分析。
* 开启_all字段，会带来额外的CPU开销和存储，如果没有使用到，可以关闭_all字段。

#### `store`
这个属性的作用是决定一个字段是否要被store，大家可能会有疑问，_source里面不是已经存储了原始的文档嘛，为什么还需要一个额外的store属性呢？原因如下：
* 如果禁用了_source保存，可以通过指定store属性来单独保存某个或某几个字段，而不是将整个输入文档保存到_source中。
* 如果_source中有长度很长的文本（如一篇文章）和较短的文本（如文章标题），当只需要取出标题时，如果使用_source字段，ES需要读取整个_source字段，然后返回其中的title，由此会引来额外的IO开销，降低效率。此时可以选择将title的store设置为true，在_source字段外单独存储一份。读取时不必在读取整个_source字段了。

但是需要注意，应该避免使用store查询多个字段，因为store的存储在磁盘上不连续，ES在读取不同的store值时，每个字段的读取均需要在磁盘上进行seek操作,而使用_source字段可以一次性连续读取多个字段[[2]]。(http://elasticsearch-users.115913.n3.nabble.com/What-does-it-mean-to-quot-store-quot-a-field-td3514176.html)


#### `doc_values`
倒排索引可以提供全文检索能力，但是无法提供对排序和数据聚合的支持。doc_value采用了面向列的存储方式存储一个field的内容，可以实现高效的排序和聚合。默认情况下，ES几乎会为所有类型的字段存储doc_value，为数不多的例外类型是analyzed类型。如果不需要对某个字段进行排序或者聚合，则可以关闭该字段的doc_value存储。很多ES的功能都使用到了doc_value进行加速，因此不建议关闭。


#### `fielddata`
即将被doc_values取代，已废弃，不再深入。

#### `term_vector`
在对文本进行analyze的过程中，可以保留有关分词结果的相关信息，包括单词列表、单词之间的先后顺序、单词在原文中的位置等信息。查询结果返回的高亮信息就可以利用其中的数据来返回。默认情况下，term_vector是关闭的，如有需要（如加速highlight结果）可以开启该字段的存储。


#### `enabled`
这是一个总开关，如果enabled设置为false，则这个字段将会仅存在于_source中，其对应的index和doc_value都不会被创建。这意味着，该字段将不可以被搜索、排序或者聚合，但可以通过_source获取其原始值。


参考文献：
[[1]Elasticsearch as a column store](https://www.elastic.co/blog/elasticsearch-as-a-column-store)
[[2]What does it mean to "store" a field?](http://elasticsearch-users.115913.n3.nabble.com/What-does-it-mean-to-quot-store-quot-a-field-td3514176.html)