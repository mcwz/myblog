---
layout: post
title: MySQL 5.7 新增加的JSON特性对应的json方法
description: 新的项目需要json特性，否则就可能涉及到横、纵表转化了，因为是新的特性，以前从没有接触过，因此查了一下MySQL的文档，并大致研究了一下对应的方法，这里记录一下。
keywords: MySQL,JSON,function,方法
---

新项目要设置一个可以动态添加各种字段的属性，一直不想直接以修改数据表列的形式来实现，不仅不优雅，而且如果项目中的每个元素需要增加的属性数量不一致或者是完全南辕北辙的话，那么表就完全乱了。

于是想到了以前的一个项目是靠着横纵表的形式实现的，就是把不确定的每个属性转成一行行的记录，这样就相当于把数据表中不确定的列转成了一行行的记录，这么做的话，每个元素多几个属性少几个属性，无非也就是记录行数的差异，完全不影响了。但是这种方式有个弊端就是在查询的时候很不方便，需要把横表转成纵表才行。

正纠结着忽然就想起来MySQL最新版(5.7)开始支持JSON形式的数据了,这样的话，多存一列的数据，无非就是增加一个key,value了。

于是在网上搜了一下，说这方面的比较少，因此自己找到官方的说明，详细看了看，记下来，留待以后看。

MySQL 函数分为四类，分别是创建（Create JSON Values）、修改（Modify JSON Values）、查询（Search JSON Values）以及返回json相关属性（Return JSON Value Attributes）的方法。

## 创建类的方法

### JSON_ARRAY([val[, val] ...])

创建JSON数组形式的数据

<pre class="prettyPrint">

SELECT JSON_ARRAY(1, "abc", NULL, TRUE, CURTIME());

[1, "abc", null, true, "11:30:24.000000"] 

</pre>

### JSON_OBJECT([key, val[, key, val] ...])

创建一个JSON对象

<pre class="prettyPrint">

SELECT JSON_OBJECT('id', 87, 'name', 'carrot');

 {"id": 87, "name": "carrot"}   

</pre>

### 修改类的方法

### JSON_ARRAY_APPEND(json_doc, path, val[, path, val] ...)

在JSON数组后增加新的数据

<pre class="prettyPrint">

mysql> SET @j = '["a", ["b", "c"], "d"]';

mysql> SELECT JSON_ARRAY_APPEND(@j, '$[1]', 1);

["a", ["b", "c", 1], "d"]

</pre>


### JSON_ARRAY_INSERT(json_doc, path, val[, path, val] ...)

插入数据到JSON中，path为插入的路径


<pre class="prettyPrint">

mysql> SET @j = '["a", {"b": [1, 2]}, [3, 4]]';
mysql> SELECT JSON_ARRAY_INSERT(@j, '$[1]', 'x');

["a", "x", {"b": [1, 2]}, [3, 4]]

</pre>


### JSON_MERGE(json_doc, json_doc[, json_doc] ...)

合并JSON数据

<pre class="prettyPrint">

SELECT JSON_MERGE('[1, 2]', '[true, false]');

[1, 2, true, false]

</pre>


### JSON_REMOVE(json_doc, path[, path] ...)

删除一部分JSON数据

<pre class="prettyPrint">

mysql> SET @j = '["a", ["b", "c"], "d"]';
mysql> SELECT JSON_REMOVE(@j, '$[1]');

["a", "d"]  

</pre>

### JSON_REPLACE(json_doc, path, val[, path, val] ...)

替换一部分JSON数据

<pre class="prettyPrint">

mysql> SET @j = '{ "a": 1, "b": [2, 3]}';
mysql> SELECT JSON_REPLACE(@j, '$.a', 10, '$.c', '[true, false]');

{"a": 10, "b": [2, 3]}  

</pre>

### JSON_SET(json_doc, path, val[, path, val] ...)

有存在的数据就替换，没有就插入

<pre class="prettyPrint">

mysql> SET @j = '{ "a": 1, "b": [2, 3]}';
mysql> SELECT JSON_SET(@j, '$.a', 10, '$.c', '[true, false]');

{"a": 10, "b": [2, 3], "c": "[true, false]"} 

</pre>


## 查询类方法

### JSON_CONTAINS(json_doc, val[, path])

查找是否包含

<pre class="prettyPrint">

mysql> SET @j = '{"a": 1, "b": 2, "c": {"d": 4}}';
mysql> SET @j2 = '1';
mysql> SELECT JSON_CONTAINS(@j, @j2, '$.a');

1

mysql> SELECT JSON_CONTAINS(@j, @j2, '$.b');

0

</pre>


### JSON_CONTAINS_PATH(json_doc, one_or_all, path[, path] ...)

查找path（一般就是key）是否存在

<pre class="prettyPrint">

mysql> SET @j = '{"a": 1, "b": 2, "c": {"d": 4}}';
mysql> SELECT JSON_CONTAINS_PATH(@j, 'one', '$.a', '$.e');

1

mysql> SELECT JSON_CONTAINS_PATH(@j, 'all', '$.a', '$.e');

0

</pre>


### JSON_EXTRACT(json_doc, path[, path] ...)

分解JSON 并查询，实际上就是在提供的path下查找值

<pre class="prettyPrint">

mysql> SELECT JSON_EXTRACT('[10, 20, [30, 40]]', '$[1]');

 20
 
 
mysql> SELECT JSON_EXTRACT('[10, 20, [30, 40]]', '$[1]', '$[0]');

[20, 10]

</pre>


### JSON_EXTRACT 的替代语法 column->path

以下两种方式等价

<pre class="prettyPrint">

mysql> SELECT c, JSON_EXTRACT(c, "$.id"), g 
     > FROM jemp
     > WHERE JSON_EXTRACT(c, "$.id") > 1 
     > ORDER BY JSON_EXTRACT(c, "$.name");
	 
mysql> SELECT c, c->"$.id", g 
     > FROM jemp
     > WHERE c->"$.id" > 1 
     > ORDER BY c->"$.name";

</pre>


### JSON_KEYS(json_doc[, path])

提出当前提供path下的key值

<pre class="prettyPrint">

SELECT JSON_KEYS('{"a": 1, "b": {"c": 30}}');

["a", "b"]

</pre>


### JSON_SEARCH(json_doc, one_or_all, search_str[, escape_char[, path] ...])

按着提供的值去查询，返回path数组

<pre class="prettyPrint">

mysql> SET @j = '["abc", [{"k": "10"}, "def"], {"x":"abc"}, {"y":"bcd"}]';

mysql> SELECT JSON_SEARCH(@j, 'one', 'abc');

 "$[0]"     
 
 
 mysql> SELECT JSON_SEARCH(@j, 'all', 'abc');
 
 ["$[0]", "$[2].x"] 

</pre>


## 查询JSON自有属性的方法

### JSON_DEPTH(json_doc)

查询当前JSON深度

<pre class="prettyPrint">

SELECT JSON_DEPTH('[10, {"a": 20}]');

3

</pre>



### JSON_LENGTH(json_doc[, path])

查询当前层级(path)下对象或者数组的元素数量

<pre class="prettyPrint">

mysql> SELECT JSON_LENGTH('{"a": 1, "b": {"c": 30}}');

2

</pre>


### JSON_TYPE(json_val)

返回JSON值类型

<pre class="prettyPrint">

mysql> SET @j = '{"a": [10, true]}';
mysql> SELECT JSON_TYPE(@j);

 OBJECT  
 
SELECT JSON_TYPE(JSON_EXTRACT(@j, '$.a'));
 
ARRAY
 
</pre>


### JSON_VALID(val)

验证是否是JSON

<pre class="prettyPrint">

mysql> SELECT JSON_VALID('{"a": 1}');

1

SELECT JSON_VALID('hello'), JSON_VALID('"hello"');

0,1

</pre>



以上是基本的方法。

后续有机会，在PHP上做些实验的话，在详细分享一下。

注：以上内容参考自官方文档：https://dev.mysql.com/doc/refman/5.7/en/json-functions.html

说的不是很详细，详细的话请参考官方文档。
