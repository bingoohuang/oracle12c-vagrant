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
