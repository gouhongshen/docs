
1. **The Current Implementation**
   
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

    after the user launches `show accounts`, mo starts a transaction and steps into a loop to execute SQL to get table info for each account. 

    built-in functions `mo_table_rows` and `mo_table_size` have been used by these functions, which first subscribes tables from TN, and then calculates rows and size from block metadata and in-memory writes.

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
        mo_catalog.mo_user AS mu2 ON /* join condition for mu2 */
    WHERE
        mt.relkind != '%s'
        AND mt.account_id = %d;
    
    ```

    **How much does the ****`show accounts`**** cost?**

    the cost depends on the number of accounts and how many tables an account has:
      1. the `join` operation could be time-consuming
      2. massive subscriptions could degrade CN performance

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
    
**Implementation Detail**

so we also could record the table size and row counts changing increasingly, 
we only need to add two extra batches to ckp, like:
   
 ```Go
    type BlkAccInsBatch struct {
        // { accout_id, database_id, table_id, location }
	    Attrs   []string 
	    Vecs    []Vector
	    //...
    }

    type BlkAccDelBatch struct {
        // { accout_id, database_id, table_id, location }
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

<br/>

3. **Implementation Considerations**
    1. timeliness

       by default, will do increment ckp each minute and global ckp each 100 increment ckps have done.
   
       and ckp is also constrained by the minimal updates count restrict
       
     3. cost
        
        through this approach, can circumvent `join` and subscription, by adding a little complication to ckp.
        
    5. The trigger condition
   
        needs a new way to request cn for storage usage statistics after discarding `show accounts`.
   
        candidate: `mo_ctl`

