两个问题：
1. 耗时[问题]([[MOC BUG]: ShowAccountsLong 告警, 执行延迟超过阈值 30s: 41.73s · Issue #2564 · matrixorigin/MO-Cloud (github.com)](https://github.com/matrixorigin/MO-Cloud/issues/2564))
2. db 和 table 数量（可见性）[问题]([[Bug]: `show accounts` returns different db counts with `show databases` · Issue #13376 · matrixorigin/matrixone (github.com)](https://github.com/matrixorigin/matrixone/issues/13376))
	1. 是否统计订阅的 db，table 数量和 size 信息？
	2. 是否统计隐藏表(\_\_index__, mo_increment_columns 的数量和 size 信息？
	3. 是否只统计属于该租户的 db 和 table（是否统计 mo_database，mo_tables, mo_column 这三张表）


<br>

show accounts output
```
mysql> show accounts;
+--------------+------------+---------------------+--------+----------------+----------+-------------+------------+----------------+
| account_name | admin_name | created             | status | suspended_time | db_count | table_count | size       | comment        |
+--------------+------------+---------------------+--------+----------------+----------+-------------+------------+----------------+
| sys          | root       | 2024-02-29 03:09:25 | open   | NULL           |        8 |          88 | 918.500359 | system account |
| acc0         | root_0     | 2024-02-29 03:16:23 | open   | NULL           |        6 |          58 |  30.683949 |                |
| acc1         | root_1     | 2024-02-29 10:22:19 | open   | NULL           |        5 |          56 |   0.011271 |                |
| acc4         | root_4     | 2024-02-29 10:22:21 | open   | NULL           |        5 |          56 |   0.011274 |                |
+--------------+------------+---------------------+--------+----------------+----------+-------------+------------+----------------+
```
<br>

执行 `show account` 时有三个步骤：
1. `select from mo_catalog.mo_account` 获取所有 account 的 `account_name`, `account_id`, `created`, `status`, `suspend_time`, `comment`

2. 对每个 account 执行
	1. 切换 ctx, attach account id
	2. ``select from mo_catalog.mo_user`` for `admin_name`
	3. `select from mo_catalog.mo_tables, mo_catalog.mo_database` for `db_count`, `table_count`
```sql
	SELECT 
    ( 
		SELECT mu2.user_name AS `admin_name`
	    FROM mo_catalog.mo_user AS mu2
	    JOIN 
	    (
			SELECT min(user_id) AS `min_user_id`
		    FROM mo_catalog.mo_user ) AS mu1
		    ON mu2.user_id = mu1.min_user_id 
		) AS `admin_name`, 
        count(distinct md.datname) AS `db_count`, 
        count(distinct mt.relname) AS `table_count`, 
        cast(0 AS double) AS `size`
	    FROM mo_catalog.mo_tables AS mt, mo_catalog.mo_database AS md
	    WHERE mt.relkind != '%s' AND mt.account_id = %d;
	)
```


3. 向 TN 发起 RPC 请求，一次性获取所有 account 的 storage usage 信息
	TN 使用缓存，这一步骤耗时基本等于 RPC 耗时。

<br>

*有什么问题？*
#### 问题一：耗时问题
现在 show accounts 的耗时与 account 数量，MO负载成正比，account 越多，耗时越久，负载越高，耗时越久。

问题出在什么地方？
问题出在第 2 步获取各个 account 的 admin_name 信息上。
各个租户的 `mo_catalog.mo_user` 表是物理隔离，为了获取租户的 `admin_name` 需要切换 ctx. 一次执行需要花费 10ms~200ms，account 数量一多，负载一高，`show accounts` 耗时就高达几十秒。当前 prod 有 160+ 租户，耗时能高达 40s+

<br>

解决方法：
1. 并行获取每个 account 的 `admin_name`. 提升大致 3x~5x 速度，但 account 数量一多，依旧会非常耗时。 
2. 留后门，不切换 ctx 的情况下，给与 system account 一次性获取所有 account 的  `admin_name` 的能力
3. storage usage task 只需要 status, account_name, size 是否可以去掉 schema 里的其他字段？
4. 是否可以区别对待 internal 和 external 的 show accounts？  
5. 修改 `mo_account` 表，将 `admin_name` 字段加入其中，这样就能依靠如下的 SQL 一次性获取除 size 外的所有信息。
```sql
+----------------+---------------------+------+------+---------+-------+---------+
| Field          | Type                | Null | Key  | Default | Extra | Comment |
+----------------+---------------------+------+------+---------+-------+---------+
| account_id     | INT(32)             | NO   | PRI  | NULL    |       |         |
| account_name   | VARCHAR(300)        | YES  | UNI  | NULL    |       |         |
| admin_name     |                     |      |      |         |       |         |
| admin_id       |                     |      |      |         |       |         |
| ...                                                                            |
+----------------+---------------------+------+------+---------+-------+---------+
```

<br>

```sql
SELECT 
	ma.account_name, 
	ma.admin_name,
	ma.account_id, 
	COUNT(DISTINCT md.dat_id) as db_count, 
	COUNT(DISTINCT mt.rel_id) as tbl_count,
	...
FROM 
	mo_catalog.mo_account AS ma
	
LEFT JOIN 
	mo_catalog.mo_tables AS mt ON ma.account_id = mt.account_id
	
LEFT JOIN 
	mo_catalog.mo_database AS md ON ma.account_id = md.account_id

GROUP BY 
	ma.account_id, 
	ma.account_name;
```

<br>

#### 问题二：db，table 可见性问题
这里面又有三个子问题
1. 是否统计订阅的 db，table 数量和 size 信息？（暂时都不统计）
2. 是否统计隐藏表(\_\_index_\*, mo_increment_columns 的数量和 size 信息？（个数不需要，size 需要）
3. 是否只统计属于该租户的 db 和 table（是否统计 mo_database，mo_tables, mo_column 这三张表）（都统计）

<br>

##### 首先是订阅 db 的可见性
租户订阅一个 db 后，可以在该租户的 mo_databases 中查找到该 db 的信息，但无法在 mo_tables 中查找到相关的 table 信息。
导致 show accounts 和 Cloud 面板上的 table 数量不匹配。

问题：
	show accounts 是否需要统计订阅的 db 和 table 数量？
	如果需要，其实现也可能面临耗时的切换操作。
	如果需要，是否也要统计上 size 信息？（dn 端不太好做这个功能，现在的逻辑没有办法知道订阅了什么 db）

 <br>

##### 然后是隐藏表的可见性
当前的实现统计了隐藏表的数量和 size

<br>

##### 最后是系统租户的 db 可见性
当前的实现没有统计属于 sys 租户的这三张表，可能与 cloud 面板展示的 table 数量不一致。

<br>

#### mo_table_size 和 show acounts [统计方式]([[MOC BUG]: 数据库界面和实例界面上的存储数字还是对不上 · Issue #2541 · matrixorigin/MO-Cloud (github.com)](https://github.com/matrixorigin/MO-Cloud/issues/2541))
1. 一张表的索引表大小应当统计到该表的大小上 \_\_index_*
2. mo_table_size 上层关联一张表和他的所有索引表，然后统一统计
3. 或者 mo_table_size 加参数控制是否统计索引表

