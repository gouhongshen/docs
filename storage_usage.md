
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

    given a timestamp, checkpoint increasingly collects all dirty blocks before that time, so we also could record the table size and row counts changing increasingly, we only need to add an extra batch to ckp, like:

    ```Go
    type TableMetaCKp struct {
      entries []struct {
        accountId uint64
        databaseId uint64
        tableId uint64
        rowDelta int64
        sizeDelta int64
      }
      // ...
    }
    ```

    in a global checkpoint, it retains all storage usage info for all accounts as of the creation of that global checkpoint.

    when users want to know the storage usage of any of an account, the dn sends a list of checkpoints, composed of the global checkpoint and all increment checkpoints after it, to cn and then it calculates the final result.

<br/>

3. **Implementation Considerations**
    1. timeliness

        by default, will do increment ckp each minute and global ckp each 100 increment ckps have done.
    2. cost

        through this approach, can circumvent `join` and subscription, by adding a little complication to ckp.
    3. trigger condition

        need a new way to request cn for storage usage statistics after discarding `show accounts`. the `select mo_ctl` is a candidate, but lacks of authentication method.

