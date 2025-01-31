# ALTER TABLE APPEND<a name="r_ALTER_TABLE_APPEND"></a>

Appends rows to a target table by moving data from an existing source table\. Data in the source table is moved to matching columns in the target table\. Column order doesn't matter\. After data is successfully appended to the target table, the source table is empty\. ALTER TABLE APPEND is usually much faster than a similar [CREATE TABLE AS](r_CREATE_TABLE_AS.md) or [INSERT](r_INSERT_30.md) INTO operation because data is moved, not duplicated\. 

**Note**  
ALTER TABLE APPEND moves data blocks between the source table and the target table\. To improve performance, ALTER TABLE APPEND doesn't compact storage as part of the append operation\. As a result, storage usage increases temporarily\. To reclaim the space, run a [VACUUM](r_VACUUM_command.md) operation\.

Columns with the same names must also have identical column attributes\. If either the source table or the target table contains columns that don't exist in the other table, use the IGNOREEXTRA or FILLTARGET parameters to specify how extra columns should be managed\. 

You can't append an identity column\. If both tables include an identity column, the command fails\. If only one table has an identity column, include the FILLTARGET or IGNOREEXTRA parameter\. For more information, see [ALTER TABLE APPEND Usage Notes](#r_ALTER_TABLE_APPEND_usage)\.

Both the source table and the target table must be permanent tables\. Both tables must use the same distribution style and distribution key, if one was defined\. If the tables are sorted, both tables must use the same sort style and define the same columns as sort keys\.

An ALTER TABLE APPEND command automatically commits immediately upon completion of the operation\. It can't be rolled back\. You can't run ALTER TABLE APPEND within a transaction block \(BEGIN \.\.\. END\)\. 

## Syntax<a name="r_ALTER_TABLE_APPEND-synopsis"></a>

```
ALTER TABLE target_table_name APPEND FROM source_table_name 
[ IGNOREEXTRA | FILLTARGET ]
```

## Parameters<a name="r_ALTER_TABLE_APPEND-parameters"></a>

 *target\_table\_name*   
The name of the table to which rows are appended\. Either specify just the name of the table or use the format *schema\_name\.table\_name* to use a specific schema\. The target table must be an existing permanent table\.

 FROM *source\_table\_name*   
The name of the table that provides the rows to be appended\. Either specify just the name of the table or use the format *schema\_name\.table\_name* to use a specific schema\. The source table must be an existing permanent table\.

IGNOREEXTRA   
A keyword that specifies that if the source table includes columns that are not present in the target table, data in the extra columns should be discarded\. You can't use IGNOREEXTRA with FILLTARGET\. 

FILLTARGET   
A keyword that specifies that if the target table includes columns that are not present in the source table, the columns should be filled with the [DEFAULT](r_CREATE_TABLE_NEW.md#create-table-default) column value, if one was defined, or NULL\. You can't use IGNOREEXTRA with FILLTARGET\. 

## ALTER TABLE APPEND Usage Notes<a name="r_ALTER_TABLE_APPEND_usage"></a>

ALTER TABLE APPEND moves only identical columns from the source table to the target table\. Column order doesn't matter\. 

 If either the source table or the target tables contains extra columns, use either FILLTARGET or IGNOREEXTRA according to the following rules: 
+ If the source table contains columns that don't exist in the target table, include IGNOREEXTRA\. The command ignores the extra columns in the source table\.
+ If the target table contains columns that don't exist in the source table, include FILLTARGET\. The command fills the extra columns in the target table with either the default column value or IDENTITY value, if one was defined, or NULL\.
+ If both the source table and the target table contain extra columns, the command fails\. You can't use both FILLTARGET and IGNOREEXTRA\. 

If a column with the same name but different attributes exists in both tables, the command fails\. Like\-named columns must have the following attributes in common: 
+ Data type
+ Column size
+ Compression encoding
+ Not null
+ Sort style
+ Sort key columns
+ Distribution style
+ Distribution key columns

You can't append an identity column\. If both the source table and the target table have identity columns, the command fails\. If only the source table has an identity column, include the IGNOREEXTRA parameter so that the identity column is ignored\. If only the target table has an identity column, include the FILLTARGET parameter so that the identity column is populated according to the IDENTITY clause defined for the table\. For more information, see [DEFAULT](r_CREATE_TABLE_NEW.md#create-table-default)\. 

## ALTER TABLE APPEND Examples<a name="r_ALTER_TABLE_APPEND_examples"></a>

Suppose your organization maintains a table, SALES\_MONTHLY, to capture current sales transactions\. You want to move data from the transaction table to the SALES table, every month\. 

You can use the following INSERT INTO and TRUNCATE commands to accomplish the task\. 

```
insert into sales (select * from sales_monthly);
truncate sales_monthly;
```

However, you can perform the same operation much more efficiently by using an ALTER TABLE APPEND command\. 

First, query the [PG\_TABLE\_DEF](r_PG_TABLE_DEF.md) system catalog table to verify that both tables have the same columns with identical column attributes\. 

```
select trim(tablename) as table, "column", trim(type) as type,
encoding, distkey, sortkey, "notnull" 
from pg_table_def where tablename like 'sales%';

table      | column     | type                        | encoding | distkey | sortkey | notnull
-----------+------------+-----------------------------+----------+---------+---------+--------
sales      | salesid    | integer                     | lzo      | false   |       0 | true   
sales      | listid     | integer                     | none     | true    |       1 | true   
sales      | sellerid   | integer                     | none     | false   |       2 | true   
sales      | buyerid    | integer                     | lzo      | false   |       0 | true   
sales      | eventid    | integer                     | mostly16 | false   |       0 | true   
sales      | dateid     | smallint                    | lzo      | false   |       0 | true   
sales      | qtysold    | smallint                    | mostly8  | false   |       0 | true   
sales      | pricepaid  | numeric(8,2)                | delta32k | false   |       0 | false  
sales      | commission | numeric(8,2)                | delta32k | false   |       0 | false  
sales      | saletime   | timestamp without time zone | lzo      | false   |       0 | false   
salesmonth | salesid    | integer                     | lzo      | false   |       0 | true   
salesmonth | listid     | integer                     | none     | true    |       1 | true   
salesmonth | sellerid   | integer                     | none     | false   |       2 | true   
salesmonth | buyerid    | integer                     | lzo      | false   |       0 | true   
salesmonth | eventid    | integer                     | mostly16 | false   |       0 | true   
salesmonth | dateid     | smallint                    | lzo      | false   |       0 | true   
salesmonth | qtysold    | smallint                    | mostly8  | false   |       0 | true   
salesmonth | pricepaid  | numeric(8,2)                | delta32k | false   |       0 | false  
salesmonth | commission | numeric(8,2)                | delta32k | false   |       0 | false  
salesmonth | saletime   | timestamp without time zone | lzo      | false   |       0 | false
```

Next, look at the size of each table\.

```
select count(*) from sales_monthly;
 count
-------
  2000
(1 row)

select count(*) from sales;
 count
-------
 412,214
(1 row)
```

Now execute the following ALTER TABLE APPEND command\.

```
alter table sales append from sales_monthly;         
```

Look at the size of each table again\. The SALES\_MONTHLY table now has 0 rows, and the SALES table has grown by 2000 rows\.

```
select count(*) from sales_monthly;
 count
-------
     0
(1 row)

select count(*) from sales;
 count
-------
 414214
(1 row)
```

If the source table has more columns than the target table, specify the IGNOREEXTRA parameter\. The following example uses the IGNOREEXTRA parameter to ignore extra columns in the SALES\_LISTING table when appending to the SALES table\.

```
alter table sales append from sales_listing ignoreextra;
```

If the target table has more columns than the source table, specify the FILLTARGET parameter\. The following example uses the FILLTARGET parameter to populate columns in the SALES\_REPORT table that don't exist in the SALES\_MONTH table\.

```
alter table sales_report append from sales_month filltarget;
```