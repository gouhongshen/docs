### background

#### `show accounts` output
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
  
#### 执行 `show account` 时有三个步骤：
1. `select from mo_catalog.mo_account` 获取所有 account 的 `account_name`, `created`, `status`, `suspend_time`, `comment`

`mo_account` 结构
```
+----------------+---------------------+------+------+---------+-------+---------+
| Field          | Type                | Null | Key  | Default | Extra | Comment |
+----------------+---------------------+------+------+---------+-------+---------+
| account_id     | INT(32)             | NO   | PRI  | NULL    |       |         |
| account_name   | VARCHAR(300)        | YES  | UNI  | NULL    |       |         |
| status         | VARCHAR(300)        | YES  |      | NULL    |       |         |
| created_time   | TIMESTAMP(0)        | YES  |      | NULL    |       |         |
| comments       | VARCHAR(256)        | YES  |      | NULL    |       |         |
| suspended_time | TIMESTAMP(0)        | YES  |      | null    |       |         |
+----------------+---------------------+------+------+---------+-------+---------+
```

2. 对每个 account 执行
	1. 切换 ctx
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


`mo_user` 结构
```
+-----------------------+--------------+------+------+---------+-------+---------+
| Field                 | Type         | Null | Key  | Default | Extra | Comment |
+-----------------------+--------------+------+------+---------+-------+---------+
| user_id               | INT(32)      | NO   | PRI  | NULL    |       |         |
| user_host             | VARCHAR(100) | YES  |      | NULL    |       |         |
| user_name             | VARCHAR(300) | YES  | UNI  | NULL    |       |         |
| authentication_string | VARCHAR(100) | YES  |      | NULL    |       |         |
| status                | VARCHAR(8)   | YES  |      | NULL    |       |         |
| created_time          | TIMESTAMP(0) | YES  |      | NULL    |       |         |
| expired_time          | TIMESTAMP(0) | YES  |      | NULL    |       |         |
| login_type            | VARCHAR(16)  | YES  |      | NULL    |       |         |
| creator               | INT(32)      | YES  |      | NULL    |       |         |
| owner                 | INT(32)      | YES  |      | NULL    |       |         |
| default_role          | INT(32)      | YES  |      | NULL    |       |         |
+-----------------------+--------------+------+------+---------+-------+---------+
```

mo_tables 结构
```
+-----------------+--------------------+------+------+---------+-------+---------+
| Field           | Type               | Null | Key  | Default | Extra | Comment |
+-----------------+--------------------+------+------+---------+-------+---------+
| rel_id          | BIGINT UNSIGNED(0) | YES  | PRI  | NULL    |       |         |
| relname         | VARCHAR(5000)      | YES  |      | NULL    |       |         |
| reldatabase     | VARCHAR(5000)      | YES  |      | NULL    |       |         |
| reldatabase_id  | BIGINT UNSIGNED(0) | YES  |      | NULL    |       |         |
| relkind         | VARCHAR(5000)      | YES  |      | NULL    |       |         |
| created_time    | TIMESTAMP(0)       | YES  |      | NULL    |       |         |
| creator         | INT UNSIGNED(0)    | YES  |      | NULL    |       |         |
| owner           | INT UNSIGNED(0)    | YES  |      | NULL    |       |         |
| account_id      | INT UNSIGNED(0)    | YES  |      | NULL    |       |         |
+-----------------+--------------------+------+------+---------+-------+---------+
```

mo_database 结构
```
+------------------+--------------------+------+------+---------+-------+---------+
| Field            | Type               | Null | Key  | Default | Extra | Comment |
+------------------+--------------------+------+------+---------+-------+---------+
| dat_id           | BIGINT UNSIGNED(0) | YES  | PRI  | NULL    |       |         |
| datname          | VARCHAR(5000)      | YES  |      | NULL    |       |         |
| owner            | INT UNSIGNED(0)    | YES  |      | NULL    |       |         |
| creator          | INT UNSIGNED(0)    | YES  |      | NULL    |       |         |
| created_time     | TIMESTAMP(0)       | YES  |      | NULL    |       |         |
| account_id       | INT UNSIGNED(0)    | YES  |      | NULL    |       |         |
+------------------+--------------------+------+------+---------+-------+---------+
```


3. 向 TN 发起请求，一次性获取所有 account 的 storage usage 信息

<br> 

#### 之前的设计解决了什么问题
最原始的版本只有前两个步骤，使用 mo_table_size 将第 3 步获取 size 嵌入到了第二步中。之前的设计主要是移除 mo_table_size，转而从 TN 低成本地获取 size 信息。


<br>
<br>


### 有什么新的问题？
现在 show accounts 的耗时与 account 数量，MO负载成正比，account 越多，耗时越久，负载越高，耗时越久。

<br>

#### 问题出在什么地方？
问题出在第 2 步获取各个 account 的 admin_name 信息上。
各个租户的 `mo_catalog.mo_user` 表是物理隔离，为了获取租户的 `admin_name` 需要切换 ctx. 一次执行需要花费 1ms~200ms，account 数量一多，负载一高，`show accounts` 耗时就高达几十秒。当前 prod 有 160+ 租户，耗时能高达 40+

之前只意识到了 mo_table_size 的复杂性，避免了 mo_table_size 的使用。
但忽视了其他简单 SQL 所累积产生的耗时。


```go
for _, id := range ids {
  //step 3.1: get the admin_name, db_count, table_count for each account
  newCtx := defines.AttachAccountId(ctx, uint32(id))

  tt = time.Now()
  if tempBatch, err = getTableStats(newCtx, bh, id); err != nil {
	  return err
	}
	getTableStatsDur += time.Since(tt)

	// step 3.2: put size value into batch
	embeddingSizeToBatch(tempBatch, usage[id], mp)

	eachAccountInfo = append(eachAccountInfo, tempBatch)
}
```

<br>
<br>

### 解决方法：
1. 并行获取每个 account 的 `admin_name`. 提升大致 3x~5x 速度，但 account 数量一多，依旧会非常耗时。 
2. 留后门，不切换 ctx 的情况下，给与 system account 一次性获取所有 account 的  `admin_name` 的能力
3. 修改 `mo_account` 表，将 `admin_name` 字段加入其中，这样就能依靠如下的 SQL 一次性获取除 size 外的所有信息。

```sql
SELECT 
	ma.account_name, 
	ma.admin_name,
	ma.account_id, COUNT(DISTINCT md.dat_id) as db_count, 
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
