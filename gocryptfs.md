# gocryptfs for oracle

## 查看表空间文件位置

1. SQL: `select FILE_NAME FROM dba_data_files;`

```sh
[oracle@localhost data]$ sqlplus / as sysdba

SQL*Plus: Release 12.2.0.1.0 Production on Thu Nov 2 06:06:30 2023

Copyright (c) 1982, 2016, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> alter session set container=ORCLPDB;

Session altered.

SQL> select * FROM dba_data_files;
FILE_NAME	FILE_ID TABLESPACE_NAME	BYTES BLOCKS STATUS    RELATIVE_FNO AUT   MAXBYTES  MAXBLOCKS INCREMENT_BY USER_BYTES USER_BLOCKS ONLINE_ LOST_WR
---- ---------- --- ---------- ---------- --------- ------------ --- ---------- ---------- ------------ ---------- ----------- ------- -------
/opt/oracle/oradata/orcl/orclpdb/system01.dbf  9 SYSTEM	262144000 32000 AVAILABLE  1 YES 3.4360E+10    4194302	   1280  261095424	 31872 SYSTEM  OFF
/opt/oracle/oradata/orcl/orclpdb/sysaux01.dbf  10 SYSAUX 377487360 46080 AVAILABLE   4 YES 3.4360E+10    4194302	   1280  376438784	 45952 ONLINE  OFF
/opt/oracle/oradata/orcl/orclpdb/undotbs01.dbf  11 UNDOTBS1 104857600 12800 AVAILABLE   9 YES 3.4360E+10    4194302	    640  103809024	 12672 ONLINE  OFF
/opt/oracle/oradata/orcl/orclpdb/users01.dbf    12 USERS 5242880 640 AVAILABLE  12 YES 3.4360E+10    4194302  160    4194304	   512 ONLINE  OFF
/opt/oracle/product/12.2.0.1.0/dbhome_1/dbs/t2.dbf  13 TBS_T2 10485760 1280 AVAILABLE  13 YES 3.4360E+10    4194302	      1    9437184	  1152 ONLINE  OFF
```

## gocryptfs 创建加密目录和绑定虚拟目录

主要命令:

1. 创建加密目录 `mkdir -p /opt/oracle/cipherdbs`
2. 创建虚拟目录 `mkdir -p /opt/oracle/plaindbs`
3. 初始加密目录 `gocryptfs -init /opt/oracle/cipherdbs`
4. 绑定虚拟目录 `gocryptfs /opt/oracle/cipherdbs /opt/oracle/plaindbs`
5. 在虚拟目录上，创建表空间：

    ```sql
    create tablespace tbs_t3 datafile '/opt/oracle/plaindbs/t3.dbf' size 10M autoextend on;
    create temporary tablespace tbs_t3_temp tempfile '/opt/oracle/plaindbs/t3_temp.dbf' size 5M autoextend on;
    create user t3 identified by "t3" default tablespace tbs_t3 temporary tablespace tbs_t3_temp;
    grant connect, resource to t3;
    grant unlimited tablespace to t3;
    ALTER USER t3 quota unlimited on tbs_t3;
    CREATE TABLE t3(id char(27) primary key, name varchar(255), addr varchar(255), email varchar(255), phone varchar(255), age varchar(3), idc varchar(256));
    insert into t3(id,name,addr,email,phone,age,idc) values('@ksuid','@姓名','@地址','@邮箱','@手机','@random_int(15-95)','@身份证');
    ```

6. godbtest 打数据 

详细日志如下：

```sh
[oracle@localhost data]$ cd /opt/oracle/
[oracle@localhost oracle]$ mkdir cipherdbs && cd cipherdbs
[oracle@localhost cipherdbs]$ gocryptfs -init .
Choose a password for protecting your files.
Password: bingoohuang
Repeat: bingoohuang

Your master key is:

    5eb752ec-02222b15-7b5e7de1-3e709b8d-
    0267b1fa-6141bcb4-b8381b24-fecb5fbc

If the gocryptfs.conf file becomes corrupted or you ever forget your password,
there is only one hope for recovery: The master key. Print it to a piece of
paper and store it in a drawer. This message is only printed once.
The gocryptfs filesystem has been created successfully.
You can now mount it using: gocryptfs . MOUNTPOINT
```

```sh
[oracle@localhost oracle]$ mkdir plaindbs
[oracle@localhost oracle]$ gocryptfs cipherdbs plaindbs
Password:
Decrypting master key
failed to unlock master key: cipher: message authentication failed
Password incorrect.
[oracle@localhost oracle]$ gocryptfs cipherdbs plaindbs
Password:
Decrypting master key
fs.Mount failed: exec: "/bin/fusermount": stat /bin/fusermount: no such file or directory
[oracle@localhost oracle]$ exit
logout
[vagrant@localhost vagrant]$ sudo yum install -y fuse
Downloading packages:
fuse-2.9.4-1.0.9.el7.x86_64.rpm                                                                                                                                                                                  |  88 kB  00:00:00
Installed:
  fuse.x86_64 0:2.9.4-1.0.9.el7

Complete!
[vagrant@localhost vagrant]$ sudo su - oracle
Last login: Thu Nov  2 06:04:40 UTC 2023 on pts/0
[oracle@localhost ~]$ cd /opt/oracle/
[oracle@localhost oracle]$ ls
admin  audit  cfgtoollogs  checkpoints  cipherdbs  diag  fast_recovery_area  oradata  plaindbs  product
[oracle@localhost oracle]$ gocryptfs cipherdbs plaindbs
Password:
Decrypting master key
fs.Mount failed: exec: "/bin/fusermount": permission denied
[oracle@localhost oracle]$ ls -hl /bin/fusermount
-rwsr-x---. 1 root fuse 32K Feb 23  2021 /bin/fusermount
[oracle@localhost oracle]$ exit
logout
[vagrant@localhost vagrant]$ sudo chmod +x /bin/fusermount
[vagrant@localhost vagrant]$ sudo su - oracle
Last login: Thu Nov  2 06:18:49 UTC 2023 on pts/0
[oracle@localhost ~]$ cd /opt/oracle/
[oracle@localhost oracle]$ gocryptfs cipherdbs plaindbs
Password:
Decrypting master key
Filesystem mounted and ready.
[oracle@localhost oracle]$
```

在 Oracle 12c 中，要让缓存的数据落盘（即将数据从内存刷新到磁盘），可以使用以下命令：

`ALTER SYSTEM FLUSH BUFFER_CACHE;`

这个命令会导致数据库将所有的脏数据块（即被修改但尚未写入磁盘的数据块）刷新到磁盘上的数据文件中。请注意，这可能会导致一些 I/O 操作和系统负载增加，因为数据库要将数据写回到磁盘上。

需要注意的几点：

- 权限要求： 执行这条命令需要足够的系统权限。一般来说，只有具有适当权限的用户或管理员才能执行这个命令。
- 谨慎使用： FLUSH BUFFER_CACHE 是一个强制刷新缓存的命令，它可能导致较大的系统开销和性能影响。一般情况下，Oracle 数据库会自行管理缓存，不需要手动干预。只有在特定情况下才需要手动刷新缓存，比如出现了严重的性能问题，或者需要强制将缓存中的数据刷新到磁盘时才会使用这个命令。

在大多数情况下，Oracle 数据库会自动管理缓存和数据写入磁盘的过程，确保数据的持久性和一致性。因此，通常情况下并不需要手动执行 FLUSH BUFFER_CACHE 这样的操作。

```sh
[oracle@localhost plaindbs]$ /vagrant/gstrings -f t3.dbf -s 18842998799
18842998799
[oracle@localhost plaindbs]$ /vagrant/gstrings -f t3.dbf -s 屈摊宴
屈摊宴;安徽省淮南市瘞蘦路5686号娥熲小区2单元946室
[oracle@localhost plaindbs]$ /vagrant/gstrings -f t3.dbf -n 15
}|{z��
$g�De%�D.~
TBS_T3
��������������������
BCCeBCCe
AAAAAAAAAAAAAAAA
2Xbo4PBt6iftrLUXkenA2EhFrhr
单怅槭;四川省眉山市铌暌路6748号憔鈦小区7单元360室
gmzuftzv@sbppd.co
18719527322
544938200811127085,
2Xbo4S7G2YCgMgnB0gAvlUyhxOl
曲粺慻<台湾省台湾省劶膐路3433号绍抐小区7单元1520室
exvvzgdi@czsqb.store
13463486716
```

## 写入对比

结论：批量写入10万数据，性能没有差异。

```sh
> %connect 'oracle://t3:t3@127.0.0.1:1521/ORCLPDB?lob fetch=post'
> truncate table t3;
+--------------+--------------+--------------+
| lastInsertId | rowsAffected | cost         |
+--------------+--------------+--------------+
|            0 |            0 | 240.219188ms |
+--------------+--------------+--------------+
> %perf -n 100000;
> insert into t3(id,name,addr,email,phone,age,idc) values('@ksuid','@姓名','@地址','@邮箱','@手机','@random_int(15-95)','@身份证')\P
2023/11/02 14:45:58 threads(goroutines): 12, txBatch: 100, batch: 1 with 100000 request(s)
2023/11/02 14:45:58 preparedQuery: insert into t3(id, name, addr, email, phone, age, idc)  values (:1, :2, :3, :4, :5, :6, :7)
100000 / 100000 [--------------------------------------------------------] 100.00% 2995 p/s 34s
2023/11/02 14:46:31 Average 335.916µs/record, total cost: 33.591641539s, total affected: 100000, errors: 0

> %connect 'oracle://t2:ca@127.0.0.1:1521/ORCLPDB?lob fetch=post'
> %perf -n 100000;
> truncate table t2;
+--------------+--------------+--------------+
| lastInsertId | rowsAffected | cost         |
+--------------+--------------+--------------+
|            0 |            0 | 165.969582ms |
+--------------+--------------+--------------+
> insert into t2(id,name,addr,email,phone,age,idc) values('@ksuid','@姓名','@地址','@邮箱','@手机','@random_int(15-95)','@身份证')\P
2023/11/02 14:48:03 threads(goroutines): 12, txBatch: 100, batch: 1 with 100000 request(s)
2023/11/02 14:48:03 preparedQuery: insert into t2(id, name, addr, email, phone, age, idc)  values (:1, :2, :3, :4, :5, :6, :7)
2023/11/02 14:48:03 error occurred: read tcp 127.0.0.1:64180->127.0.0.1:1521: read: connection reset by peer
100000 / 100000 [-------------------------------------------------------] 100.00% 2909 p/s 35s
2023/11/02 14:48:38 Average 345.784µs/record, total cost: 34.57846703s, total affected: 100000, errors: 1
>
```

```sh
[oracle@localhost oracle]$ ls -hlR plaindbs/ cipherdbs/
cipherdbs/:
total 29M
-r--------. 1 oracle oinstall  403 Nov  2 06:09 gocryptfs.conf
-r--r--r--. 1 oracle oinstall   16 Nov  2 06:09 gocryptfs.diriv
-rw-r-----. 1 oracle oinstall  28M Nov  2 06:53 kgWJgJoL5EI30oABV4NznQ
-rw-r-----. 1 oracle oinstall 5.1M Nov  2 06:31 soZLC31c5HrLXMBAEx8mrQ

plaindbs/:
total 29M
-rw-r-----. 1 oracle oinstall  28M Nov  2 06:53 t3.dbf
-rw-r-----. 1 oracle oinstall 5.1M Nov  2 06:31 t3_temp.dbf
```

## unmount

```sh
[oracle@localhost oracle]$ ls -lR plaindbs/
plaindbs/:
total 29240
-rw-r-----. 1 oracle oinstall 28647424 Nov  2 07:31 t3.dbf
-rw-r-----. 1 oracle oinstall  5251072 Nov  2 06:31 t3_temp.dbf
[oracle@localhost oracle]$ fusermount -u plaindbs
[oracle@localhost oracle]$ ls -lR plaindbs/
plaindbs/:
total 0
[oracle@localhost oracle]$
```

如果遇到 `Device or resource busy`，使用 `fuser -m /path/to/mount/point`  查看哪个进程占用了这个资源，或者强制卸载 `fusermount -uz /path/to/mount/point`

## 迁移表空间文件

[Online Move Datafile in Oracle Database 12c Release 1 (12.1)](https://oracle-base.com/articles/12c/online-move-datafile-12cr1)

- `SELECT tablespace_name, file_name FROM dba_data_files;`
- `ALTER DATABASE MOVE DATAFILE '/opt/oracle/product/12.2.0.1.0/dbhome_1/dbs/t2.dbf' TO '/opt/oracle/plaindbs/t2.dbf';`

**临时表空间文件，不能简单的移动**


- [Oracle Database 12c: Moving a datafile is now possible online](https://www.dbi-services.com/blog/oracle-database-12c-moving-a-datafile-is-now-possible-online/)

Gone are the days where you had to put a tablespace or your complete database offline in order to move or rename a datafile. Oracle 12c features a new command that allows to move datafiles when both database and datafile are online!

Up to Oracle 11.2.0.3, you had three ways to relocate a datafile:

- Put the tablespace offline
- Put the datafile offline
- Shutdown and mount the database

### Oracle 11g: Tablespace offline

Renaming a datafile by switching offline its tablespace requires four steps:

1. Switching the tablespace offline  `SQL> ALTER TABLESPACE DEMO OFFLINE NORMAL;`
2. Moving the datafile with an Operating System command `$ mv '/u01/oradata/DBTEST/demo1.dbf' '/u01/oradata/DBTEST/demo01.dbf'`
3. Changing the pointer to the data file `SQL> ALTER DATABASE RENAME FILE '/u01/oradata/DBTEST/demo1.dbf' TO '/u01/oradata/DBTEST/demo01.dbf';`
4. Switching the tablespace online `SQL> ALTER TABLESPACE DEMO ONLINE;`

There are two important drawbacks:

- The database is partially unavailable during the operation: All data registered on the tablespace that is offline is unavailable.
- The default tablespace and the default temp tablespace cannot be switched offline.

### Oracle 11g: Datafile offline

Renaming a datafile this way requires one step more (5):

1. Switching the datafile offline `SQL> ALTER DATABASE DATAFILE '/u01/oradata/DBTEST/demo1.dbf' OFFLINE;`
2. Moving the datafile with an Operating System command `$ mv '/u01/oradata/DBTEST/demo1.dbf' '/u01/oradata/DBTEST/demo01.dbf'`
3. Changing the pointer to the data file `SQL> ALTER DATABASE RENAME FILE '/u01/oradata/DBTEST/demo1.dbf' TO '/u01/oradata/DBTEST/demo01.dbf';`
4. Recovering the datafile before switching it online `SQL> ALTER DATABASE RECOVER DATAFILE '/u01/oradata/DBTEST/demo01.dbf';`
5. Switching the datafile online `SQL> ALTER DATABASE DATAFILE '/u01/oradata/DBTEST/demo01.dbf' ONLINE;`

There are two important drawbacks:

- The database is partially unavailable during the operation: All data registered on the datafile that is offline is unavailable.
- The datafile needs a media recovery since it is switched offline and SCN cannot be updated during offline time.

### Oracle 11g: Shutdown and mount the database

The ultimate way to rename datafiles is to shutdown and mount the database. It allows to alter files that cannot be altered when the database is running – such as temporary tablespace, redo log files, etc. It requires five steps:

1. Shutting down the database `SQL> SHUTDOWN IMMEDIATE;`
1. Mounting the database `SQL> STARTUP MOUNT;`
1. Moving the datafile with an Operating System command `$ mv '/u01/oradata/DBTEST/demo1.dbf' '/u01/oradata/DBTEST/demo01.dbf'`
1. Changing the pointer to the data file `SQL> ALTER DATABASE RENAME FILE '/u01/oradata/DBTEST/demo1.dbf' TO '/u01/oradata/DBTEST/demo01.dbf';`
1. Starting the database `SQL> ALTER DATABASE OPEN;`

The main drawback is simple to identify: The whole database is unavailable during the operation.
In any case, up to Oracle 11g, a maintenance window was required in order to play with datafiles.

### Oracle 12c: Alter Database Move Datafile

Starting with Oracle 12.1, you can now use the command ALTER DATABASE MOVE DATAFILE in order to rename, relocate, or copy a datafile when the datafiles or the database are online. One step only is required for any of these actions and the database remains entirely available in read and write for users, without any data loss.

You should however be aware of some rules:

By default, Oracle automatically deletes old data file after moving them and prevents the user from overwriting an existing file.
When you move a data file, Oracle first makes a copy of the datafile. Then, when the file is successfully copied, pointers to the datafile are updated and the old file is removed from the file system. This is why the operation requires twice the size of the files to be copied as free space.
Let’s have a look at some examples for the renaming or relocating of a datafile:

1) Renaming a datafile `SQL> ALTER DATABASE MOVE DATAFILE '/u01/oradata/DBTEST/demo1.dbf' TO '/u01/oradata/DBTEST/demo01.dbf';`
2) Relocating a datafile `SQL> ALTER DATABASE MOVE DATAFILE '/u01/oradata/DBTEST/demo01.dbf' TO '/u02/oradata/DBTEST/demo01.dbf';`

The available options are:

`KEEP`: to keep the old datafile, used to make a copy of the file. Note that the pointer will be updated to the new file,the  old file only remains as unused on the filesystem. Note that on Windows, independently of the fact that the KEEP option is used or not, the old data file is not automatically deleted by Oracle. The user has to delete it manually after the copy is successfully performed.

`SQL> ALTER DATABASE MOVE DATAFILE '/u01/oradata/DBTEST/demo01.dbf' TO '/u02/oradata/DBTEST/demo01.dbf' KEEP;`

`REUSE`: to overwrite an existing file.

`SQL> ALTER DATABASE MOVE DATAFILE '/u01/oradata/DBTEST/demo01.dbf' TO '/u02/oradata/DBTEST/demo01.dbf' REUSE;`

Moving a datafile online works in both ARCHIVE and NO ARCHIVE modes. You have no option to enable in order to use this feature, which is great. It does not work with an OFFLINE datafile.
On the backup & recovery side, a flashback database does not revert the operation. A datafile is definitively moved. But after moving a datafile, like for any other operation on the database structure, you will have to perform a new full backup of the database. Otherwise, you will have to move the datafile(s) back before being able to restore your database…
Finally, some files like temporary files, redo log files, and control files cannot be moved using this command, which is very helpful but can still be improved…

## 资源

1. [oracle 版本历史](https://en.wikipedia.org/wiki/Oracle_Database)
