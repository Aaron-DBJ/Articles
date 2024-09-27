# SQL执行顺序

查询语句书写顺序

<img src="https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240927134452.png" title="" alt="" width="299">

**实际执行顺序**

- 我们先执行from，join来确定表之间的连接关系，得到初步的数据

- where对数据进行普通的初步的筛选

- group by 分组

- 各组分别执行having中的普通筛选或者聚合函数筛选。

- 然后把再根据我们要的数据进行select，可以是普通字段查询也可以是获取聚合函数的查询结果，如果是集合函数，select的查询结果会新增一条字段

- 将查询结果去重distinct

- 最后合并各组的查询结果，按照order by的条件进行排序

![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240927134610.png)
