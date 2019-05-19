---
date: 2016-09-24 15:26
status: public
tags: 学习
title: Sql必知必会
---

# 1 mysql installation
1. 检查mysql是否已经安装
```shell
rpm -qa | grep mysql
```

如果安装了mysql，删除之
```shell
rpm -e ×××× --nodeps
```

2. 安装server 
```shell
rpm -i Mysql-server-××××
```

3. 启动Server
```shell
mysqld_safe &
```

4. 安装client 
```shell
rpm -i Mysql-client-×××
```

5. 设置密码
```shell
mysql_secure_installation 
```
设置root密码为admin

6. 进入mysql
```shell
mysql -uroot -padmin
```

7. 导入sql脚本
```shell
\. /usr/local/create.sql
\. /usr/local/population.sql
```  
# 2 Select 查询
1. 标准查询
```sql
select field_names from table_name where condition;
```
2. 查询列名别名
```sql 
select field_name as alias_name from table_name where conditions;
```   

3. 通配符like
    + % 多个字符通配符
    + _ 单个字符通配符
    + [abcd]只匹配单个,
    + [^abcd]不匹配其中的任何字符
    
4. 拼接 concatenate 
使用+或者||
```sql
select rtrim(vend_name) + '(' + rtrim(vend_country) +')' as vend_title from Vendors;
```  
  
# 3 aggregate function  

    + sum
    + avg
    + min
    + max 
    + count  

# 4 数据分组  

```sql
select vend_id,count(*) as num_prod from Products group by vend_id;
```  
+ group by 子句中嵌套了分组，那么数据将在最后分组上进行汇总
+ group by 在where前，order by之后
+ 过滤分组用Having 子句 

# 5 子查询
```sql  
select cust_id from orders where order_num in (select order_num from orderitems where prod_id = 'rage01') 
``` 
+ 作为子查询的select语句只能查询单列  

# 6 链接 join 
如果没有给定链接的条件，那么结果将会是笛卡尔积，检出的行数为表1的行数乘以表2的行数。cross join 
**内连接（inner join)**
```sql 
select vend_name,prod_name,prod_price from venders inner join products on vender.vend_id=products.vend_id;
```  
    
# 7 高级链接 
使用表的别名 
+ self join   

```sql 
select c1.cust_id,c1.cust_name 
from customers c1, customers c2
where c1.cust_name=c2.cust_name and c2.id='123';
```  

+ outer join 
    + left outer join  
 
```sql 
select customers.cust_id, orders.order_num
from customers left outer join orders
on customers.cust_id=orders.cust_id;
``` 
    + right outer join 
```sql 
select customers.cust_id, orders.order_num
from customers right outer join orders
on customers.cust_id=orders.cust_id;
```  
#　8 Insert table 
1. 插入整个行
```sql 
insert into tablename values(values_name); 
```  

2. 按列插入
```sql 
insert into tablename(fields) values (fields_value); 
``` 
 
3. 插入检出数据
```sql 
insert into tablename (fields) select fields from anothertablenames; 
```

4. 复制表 
```sql 
select * into newtablename from origintable;
``` 

mysql，oracle   

```sql
create table newtablename as select * from origintable;
```

# 9 Update or delete table  
1. Update table 
```sql 
update tablename 
set value 
where condition
``` 

2. Delete table 
```sql 
delete from tablename 
where condition 
``` 
3. Suggestion 
除非是删除所有行，一定要加where条件 

# 10 View 
```sql 
create view as (select sentence) 
```