# Mysql事务

---

MySQL可重复读(REPEATABLE READ)隔离级别下的幻读问题

Mysql的InnoDB存储引擎在可重复读隔离级别下默认使用**Next-key-lock**算法解决幻读问题，但可能出现并发情况下，插入数据导致主键冲突问题。

事务一：

~~~ mysql
> begin;					--- 1
> seleselect * from wow;		   --- 3
+-----+
| id  |
+-----+
|   1 |
|   2 |
|   3 |
+-----+
> insert into wow values('4');	    --- 5
Query OK, 1 row affected (0.00 sec)
~~~

事务二：

~~~ mysql
> begin;					--- 2
> seleselect * from wow;		--- 4
+-----+
| id  |
+-----+
|   1 |
|   2 |
|   3 |
+-----+
> insert into wow values('4')    --- 6
ERROR 1062 (23000): Duplicate entry '4' for key 'PRIMARY'		--- 主键冲突报错
~~~

事务一中查询表wow时没有 `id = 4`的字段，事务二中查询表wow时也没有`id = 4`的字段。随后在事务一中插入`id = 4`的字段，显示插入成功；而后在事务二中插入`id = 4`的字段时长时间没有响应，随后报了主键冲突错误。

**事务二中查询时没有`id = 4`的字段，但插入时主键冲突，无法插入 --> 是否也是一种幻读的情况**

