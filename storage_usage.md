1. **Background**

![image](https://github.com/gouhongshen/docs/assets/26336316/024759e9-e1e9-4106-8913-100b70af2b2a)


   **Where does the `show accounts` cmd come from?**
   1. internal

      the CN MetricStorageUsage cron task gathers any new accounts every minute and all accounts every 15 minutes, and then writes them in metrics.
      but this cron task only exists in exactly one CN. 
      
   3. users

      users can launch `show accounts` from the SQL client at any time.

   **Who has the access right?**
   1. system account:

      at every minute, the sys account executes `show accounts like xxx` to collect storage usage of newly added accounts.

      at every 15 minutes, the sys account executes `show accounts` to collect storage usage of all accounts
      
   3. normal account:

      any normal accounts can launch `show accounts` SQL, the effect equals to the sys account executes `show accounts like xxx`.

<br>

   **Size & Rows**
   * in memory, uncommitted
   * in memory, committed
   * persisted data

<br>
<br>

2. **The Current Implementation**
   
    using `show accounts` to get all storage usage info for all accounts.

    ```
    mysql> show accounts;
    +--------------+------------+---------------------+--------+----------------+----------+-------------+-----------+----------+----------------+
    | account_name | admin_name | created             | status | suspended_time | db_count | table_count | row_count | size     | comment        |
    +--------------+------------+---------------------+--------+----------------+----------+-------------+-----------+----------+----------------+
    | tenant_test  | admin_1    | 2023-10-06 02:43:34 | open   | NULL           |        5 |          45 |      1157 |    0.066 |                |
    | tenant_2     | admin_2    | 2023-10-06 02:44:21 | open   | NULL           |        5 |          45 |      1157 |    0.066 |                |
    | tenant_3     | admin_3    | 2023-10-06 02:44:33 | open   | NULL           |        5 |          45 |      1157 |    0.066 |                |
    | sys          | root       | 2023-09-28 09:44:41 | open   | NULL           |        7 |          82 |  32351824 | 3555.283 | system account |
    +--------------+------------+---------------------+--------+----------------+----------+-------------+-----------+----------+----------------+
    4 rows in set (0.03 sec)
    ```

    **How does ****`show accounts`**** work?**

    after the frontend receives `show accounts`, it starts a transaction.

    the first step is to get related account info:
   
   ```SQL
   SELECT
   	account_id AS `account_id`,
    	account_name AS `account_name`,
    	created_time AS `created`,
    	status AS `status`,
    	suspended_time AS `suspended_time`,
    	comments AS `comment`
   FROM
    	mo_catalog.mo_account %s;
   ``` 

    for every account, it uses the built-in functions `mo_table_rows` and `mo_table_size` to get account storage usage info, and then calculates the results with in-memory writes to get the final result.

    ```SQL
    SELECT
        mu2.user_name AS `admin_name`,
        COUNT(DISTINCT mt.reldatabase) AS `db_count`,
        COUNT(DISTINCT mt.relname) AS `table_count`,
        SUM(mo_table_rows(mt.reldatabase, mt.relname)) AS `row_count`,
        CAST(SUM(mo_table_size(mt.reldatabase, mt.relname)) / 1048576 AS DECIMAL(29, 3)) AS `size`
    FROM
        mo_catalog.mo_tables AS mt
    JOIN
        (
            SELECT
                MIN(user_id) AS `min_user_id`
            FROM
                mo_catalog.mo_user
        ) AS mu1 ON mu2.user_id = mu1.min_user_id
    JOIN
        mo_catalog.mo_user AS mu2 ON 
    WHERE
        mt.relkind != '%s' AND mt.account_id = %d;
    ```
<br>

**How much does the `show accounts` cost?**

    the cost depends on the number of accounts and how many tables an account has:
      1. the `join` operation could be time-consuming
      2. massive subscriptions could degrade CN performance
   ```vim
   mo_table_size
   mo_table_rows
    |
    |
    |-- tb_1  --
    |-- tb_2     \
    |-- tb_3       ====> UpdateBlockInfos ===> SubscribeTable
    |            /
    |-- tb_n  --
   ```

<br/>

2. **The Optimization Plan**

    in summary, using the checkpoint process, we can save CN's performance.
    <br>
    <br>

    **Checkpoint Overview**
    ```vim
    catalog
    |
    ---------------------------------------------------------
            |               |               |       ...
         db_entry        db_entry        db_entry
                           |
                           |
                          ----------------------------------------
                                    |        |         |     ...
                                tb_entry  tb_entry  tb_entr
                                    |
                                    |
                                   --------------------------------
                                            |         |        ...
                                        seg_entry  seg_entry
                                            |
                                            |
                     ------------------------------------------------------------
                            |       |         |       |        |      |       |
                           blk     blk       blk     blk      blk    blk     blk
                       |---------------|    |-----------|    |--------------------|
                       s1    ckp1     e2    s2   ckp2   e2   s3      ckp3        e3
               |----------------------------------------|
               -∞             global ckp                e2
    ```

Whenever a block is created or deleted, its metadata is recorded in the catalog. 
The checkpoint runner periodically collects this metadata and logs it in the checkpoint.

In incremental checkpoints, there are very few pieces of metadata that are empty because 
some block data has not been flushed (logged in the log service). When combined with the global checkpoint, 
the metadata that was not accounted for promptly is further reduced.

<br>

**Implementation Detail**

so we also could record the table size and row counts changing increasingly, 
we only need to add two extra batches to ckp, like:
   
 ```Go
    type BlkAccInsBatch struct {
        // { accout_id, database_id, table_id, rows, size }
	    Attrs   []string 
	    Vecs    []Vector
	    //...
    }

    type BlkAccDelBatch struct {
        // { accout_id, database_id, table_id, rows, size }
	    Attrs   []string 
	    Vecs    []Vector
	    //...
    }
```    

when the user wants to know its storage usage, it requests CN --> TN, 
and TN returns a global ckp and a list of incremental checkpoints. 
the CN needs to decode these batches stored in checkpoints and combine them with the writes in memory.
the codes like:
```python
for ckp in [global_ckp, incremental_ckps] {
    for ins_bat in ckp {
        size[ins_bat.acc_id] += ins_bat.location.length
        rows[ins_bat.acc_id] += ins_bat.location.rows
    }
    
    for del_bat in ckp {
        size[del_bat.acc_id] -= del_bat.location.length
        rows[del_bat.acc_id] -= del_bat.location.rows
    }
}
    
for data in memory_writes {
    if data.type == del {
        continue
    }
    size[data.acc_id] += data.location.length
    rows[data.acc_id] += data.location.rows
}    
    
```
<br>

**Interface Compaitable**:

result interface
```Go
// res batch
type Batch struct {
    // account_name     
    // admin_name	
    // created          
    // status           
    // suspendedTime    
    // comment
    // db_count	
    // table_account	// join mo_user, mo_tables, mo_account

    // row_count	// comes from ckp
    // size (MB)	// comes from ckp
    Attrs []string

    Vecs  []*vector.Vector
}
```

**Handle Pipeline**
* we can use the logtail pull pipeline
  ```
  OpCode_OpGetLogTail

  type SyncLogTailResp struct {
    CkpLocation
    Commands
  }
  ```
* we can use the debug pipeline (dedicated for mo_ctl)
```
TxnMethod_DEBUG
```

<br/>

3. **Others Considerations**
    1. timeliness

       by default, will do increment ckp each minute and global ckp each 100 increment ckps have done.
   
       and ckp is also constrained by the minimal updates count restrict

    2. cost
        
        * time: 1) decode serval ckps; 2) query mo_catalog.mo_user table.
        * space: need extra 4B + 8B + 8B + 4B + 4B = 28B for each blk update record in ckp

    3. accurate

       incremental ckp may be missing a few block update records since these blocks have not flushed.
       

