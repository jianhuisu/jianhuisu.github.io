---
title : "group by结果集的选择问题"
categories : 
 - mysql 
tags :
	- MySQL
---

## group by 

group by 时默认会选取第一条匹配的信息展示在最终结果集中
但是有时候需要选取最后一条记录在结果集中展示,这就要求排序要在 group by 之前

	select count(1) from (select * from table_name order by ... limit 1000) ... group by ... 
	
在子查询中排序时,如果后面不接`limit`子句,排序操作会被msyql的优化器优化掉
	
## 参考资料

https://www.v2ex.com/t/379352