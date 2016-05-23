---
title: "Introduction to Oracle Supplemental Logging for Logical Replication"
categories:
- oracle
- logical-replication
keywords: oracle, replication, supplemental logging
summary: "Supplemental logging is a pre-requisite requirement for all redo log based logical replication products that I'm familiar with, and here's why."
thumb: Puzzle_pieces_118x107.jpg
---

![Puzzle Pieces](/images/logical-replication/Puzzle_pieces_118x107.jpg)

The contents of the redo log was never intended to be used for anything other than recovery (instance or media) on the database that generated it, or in the case of physical replication like Oracle Data Guard, on a binary copy of the database.

By default, there is not enough information in the redo logs to reconstruct the SQL statements such that they can be executed in another, un-related, database. For example, the rowid of a record is database specific and can't be used in logical replication.

Additional information needs to be included with the transactions recorded in the redo log so that products like GoldenGate, Dbvist Replicate, and Dell SharePlex and Oracle procedures like Streams and Log Miner can operate.

So, supplemental logging is the addition of extra information in the redo logs. What information can be included and how it's accomplished is discussed below.

This article is not a full discussion on all aspects of Oracle Supplemental Logging. Instead, it focuses on supplemental logging as it relates to logical replication. If you're interested in a more complete discussion of Oracle supplemental logging, please see the Resources section at the end of this article.

In future posts, I'll be looking at how the three logical replication products I'm familiar with, namely Oracle GoldenGate, Dbvisit Replicate, and Dell SharePlex use Oracle supplemental logging.

Functional descriptions and behaviour in this article are those observed using the OTN Developer Day pre-built Oracle VirtualBox virtual machine (Oracle_DB_Developer_VM_new.ova) which includes Oracle Database 12c Enterprise Edition (12.1.0.2.0).

There are two levels at which supplemental logging can be enabled:

- Database Level
- Table Level

{{site.data.alerts.note}}
GoldenGate offers the ability to enable supplemental logging at the schema level through the use of it's ADD SCHEMATRANDATA command which issues calls to the Oracle Streams DBMS_CAPTURE_ADM.PREPARE_SCHEMA_INSTANTIATION procedure.
{{site.data.alerts.end}}

## Database Level Supplemental Logging

There are three types of supplemental logging at the database level.

- Minimal Supplemental Logging
- Supplemental Key Logging
- Procedural Replication Supplemental Logging

I will not be discussing Procedural Replication Supplemental Logging as I'm not aware of any logical replication products that use this type of supplemental logging.

The simplified syntax of the `ALTER DATABASE` statement, as it relates to supplemental logging for logical replication is:

![Alter Database](/images/logical-replication/alter_database.png)

### Minimal Supplemental Logging

Most replication products require minimal supplemental logging at the database level. ALTER DATABASE ADD SUPPLEMENTAL LOG DATA sets minimal supplemental logging at the database level and sets the SUPPLEMENTAL_LOG_DATA_MIN column of V$DATABASE to YES.

### Supplemental Key Logging

Supplemental Key Logging will log the columns specified by the syntax. Of interest to logical replication would be the ALL, PRIMARY KEY, and UNIQUE options. Some products use pne or more of these options, some do not.

Minimal Supplemental Logging is a pre-requisite for any level of supplemental key logging. If you set, for example, your Primary Key supplemental logging by executing the statement:

ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS
You will see the value of V$DATABASE.SUPPLEMENTAL_LOG_DATA_PK set to YES and the value of V$DATABASE.SUPPLEMENTAL_LOG_DATA_MIN set to IMPLICIT meaning that all or a combination of primary key, unique key, or foreign key supplemental key logging has been enabled.

Supplemental key logging at the database level is a relatively expensive option as you are asking the database to log the primary/unique/foreign keys, or even all columns, for every update, regardless of whether the table is being replicated.

Data Dictionary Views for Database Level Supplemental Logging Information

For databases <= 11gR2 and non-multitenant 12c databases, use the SUPPLEMENTAL_LOG_DATA columns from the V$DATABASE view.

```
SELECT supplemental_log_data_min MIN,
supplemental_log_data_pk PK,
supplemental_log_data_ui UI,
supplemental_log_data_fk FK,
supplemental_log_data_all "ALL"
FROM v$database;
```

In a multi-tenant Oracle Database 12c database there is a new dictionary view called DBA_SUPPLEMENTAL_LOGGING that provides information about supplemental logging for a pluggable database (PDB) in a multitenant container database (CDB). Use this to determine the supplemental logging level in your 12c PDB.
In a 12c multi-tenant database, the SUPPLEMENTAL_LOG_DATA columns from the V$DATABASE view relate to the CDB and all PDBs.
A value of YES or IMPLICIT in any of the SUPPLEMENTAL_LOG_DATA columns in V$DATABASE of a 12c multi-tenant database indicates that supplemental logging has been enabled in all of the PDBs in the CDB.

```
SQL> desc DBA_SUPPLEMENTAL_LOGGING
 Name					                         Null?    Type
 ----------------------------------------- -------- ----------------------------
 MINIMAL					                               VARCHAR2(3)
 PRIMARY_KEY					                         VARCHAR2(3)
 UNIQUE_INDEX					                         VARCHAR2(3)
 FOREIGN_KEY					                         VARCHAR2(3)
 ALL_COLUMN					                            VARCHAR2(3)
 PROCEDURAL					                            VARCHAR2(3)
``` 
 
## Table Level Supplemental Logging

There are two types of supplemental logging at the table level.

- Supplemental Log Groups
- Supplemental Key Column Logging.

Supplemental Log Groups use the `supplemental_log_grp_clause` syntax while Supplemental Key Column Logging use the `supplemental_id_key_clause` syntax.

![Alter Table](/images/logical-replication/alter_table.png)


### Supplemental Log Groups

When defining a supplemental log group for a table, you explicitly specify a list of columns you want logged into the redo. Replication products typically use the `ALWAYS` keyword to create an un-conditional log group which means the columns are always logged, even if they are not in the original SQL statement.


### Supplemental Key Column Logging

Supplemental Key Column Logging is also called Supplemental Data Logging in certain documentation and logs at the table level what Supplemental Logging does at the database level, but being at the table level is more selective.

Data Dictionary Views for Table Level Supplemental Logging Information

The two views are

- DBA_LOG_GROUPS
- DBA_LOG_GROUP_COLUMNS

They provide a description of the log groups and their columns as well as the supplemental key column logging.

Both supplemental log groups and supplemental key column logging information is available through the `DBA_LOG_GROUPS` view. The `LOG_GROUP_TYPE` column will be `USER LOG GROUP` for supplemental log groups and it will be one of `PRIMARY KEY LOGGING`, `UNIQUE KEY LOGGING`, or `ALL COLUMN LOGGING`.

Note that the columns logged as a result of Supplemental Key Column logging are not shown in the `DBA_LOG_GROUP_COLUMNS` view. Only those columns specified when a Supplemental Log Group is defined are shown, so a query similar to the one below can be used to display information about both types of table level supplemental logging.

If your source database is >= 11gR2 you can use the following query to display the information in a tidy format.

```
COLUMN owner FORMAT a20
COLUMN log_group_type FORMAT a20
COLUMN column_list FORMAT a60

SELECT g.owner, g.log_group_name, g.table_name, g.log_group_type,
LISTAGG(c.column_name, ', ') WITHIN GROUP (ORDER BY c.POSITION) column_list
FROM dba_log_groups g,
dba_log_group_columns c
WHERE g.owner = c.owner(+)
AND g.log_group_name = c.log_group_name(+)
AND g.table_name = c.table_name(+)
GROUP BY g.owner, g.log_group_name, g.table_name, g.log_group_type
ORDER BY g.owner, g.log_group_name, g.table_name, g.log_group_type
```

Or for 10g, try:

```
set lines 132
set pages 60

COLUMN log_group_type FORMAT a20
COLUMN column_name FORMAT a30

BREAK ON table_name ON log_group_name ON log_group_type SKIP 1

SELECT g.table_name, g.log_group_name, g.log_group_type, c.column_name
FROM dba_log_groups g,
     dba_log_group_columns c
WHERE g.owner = c.owner(+)
AND g.log_group_name = c.log_group_name(+)
AND g.table_name = c.table_name(+)
ORDER BY g.table_name, g.log_group_name, g.log_group_type, c.position
```

And remember, supplemental logging is only required on the source database for logical replication. If you have used any target instantiation methods that have copied the source object definitions to the target, don't forget to remove the supplemental logging. It's an overhead you don't need on the target.

## Resources

There are lots of good articles on the Internet regarding supplemental logging and as it relates to forms of logical replication. As always, I encourage you to read the official Oracle documentation on the topics first.

Julian Dyke has an excellent, in depth look, at supplemental logging  as it relates to GoldenGate [here](//www.juliandyke.com/Research/GoldenGate/GoldenGateSupplementalLogging.php).





