# oracle12c-vagrant

A vagrant box that provisions Oracle 12c automatically, using only Vagrant and a shell script. Forked from [repo](https://github.com/steveswinsburg/oracle12c-vagrant)

## Getting started

1. Clone this repository
2. Download [Oracle Database 12.2.0.1.0.0](https://edelivery.oracle.com/osdc/faces/SoftwareDelivery) and unzip to `database/`
  * Ensure you unzip into the same directory and they merge. There are some common directories in each. On the commandline, you can run `unzip '*.zip'` to do this for you.
3. Install Virtualbox
4. Install Vagrant
5. Install VirtualBox plugin `vagrant plugin install vagrant-vbguest`
6. Run `vagrant up`
  * The first time you run this it will provision everything and may take a while. Ensure you have a good internet connection!
7. Connect to the database (see below).
8. You can shutdown the box via the usual `vagrant halt` and the start it up again via `vagrant up`.

## Connecting to Oracle

- godbtest `%connect 'oracle://t2:ca@127.0.0.1:1521/ORCLPDB?lob fetch=post'`
  
```sql
CREATE TABLE t2 (id char(27)  primary key, name varchar(255), addr varchar(255), email varchar(255), phone varchar(255), age varchar(3), idc varchar(256));
insert into t2(id, name,addr,email,phone,age,idc) values('@ksuid', '@姓名', '@地址', '@邮箱', '@手机', '@random_int(15-95)', '@身份证');
```

## Tablespaces

The folder `oradata` is mounted as a shared folder with permissions for Oracle to use it. If you have Oracle schemas that will consume a lot of space, create a tablespace for your schema in this directory instead of using the built in tablespaces. See [tablespace.sql](/scripts/tablespace.sql) for an example of how to create a tablespace in this directory.

## Other info

* If you need to, you can connect to the machine via `vagrant ssh`.
* You can `sudo su - oracle` to switch to the oracle user.
* The Oracle installation path is `/opt/oracle/`
* On the guest OS, the directory `/vagrant` is a shared folder and maps to wherever you have this file checked out.

## log

```sh
[oracle@localhost ~]$ sqlplus / as sysdba

SQL*Plus: Release 12.2.0.1.0 Production on Thu Nov 2 02:21:12 2023

Copyright (c) 1982, 2016, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 ORCLPDB                        MOUNTED
         
SQL> alter session set container=ORCLPDB;

Session altered.

SQL> startup;

Pluggable Database opened.

# 保存状态 alter pluggable database all save state;

SQL> create tablespace tbs_t2 datafile 't2.dbf' size 10M autoextend on;

Tablespace created.

SQL> create temporary tablespace tbs_t2_temp tempfile 't2_temp.dbf' size 5M autoextend on;

Tablespace created.

SQL> create user t2 identified by "ca" default tablespace tbs_t2 temporary tablespace tbs_t2_temp;             

User created.

SQL> grant connect, resource to t2;

Grant succeeded.

SQL> grant unlimited tablespace to t2;

Grant succeeded.

SQL> ALTER USER t2 quota unlimited on tbs_t2;

User altered.

SQL> CREATE OR REPLACE TRIGGER open_pdbs
  AFTER STARTU  2  P ON DATABASE
BEGIN
   EXECUTE IMMEDIATE 'ALTER PL  3    4  UGGABLE DATABASE ALL OPEN';
END open_pdbs;
/  5    6  

Trigger created.

SQL> exit
Disconnected from Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production
```

```sh
$ godbtest
> %connect 'oracle://t2:ca@127.0.0.1:1521/ORCLPDB?lob fetch=post'
> CREATE TABLE t2 (id char(27)  primary key, name varchar(255), addr varchar(255), email varchar(255), phone varchar(255), age varchar(3), idc varchar(256));
+--------------+--------------+-------------+
| lastInsertId | rowsAffected | cost        |
+--------------+--------------+-------------+
|            0 |            0 | 109.57724ms |
+--------------+--------------+-------------+
> insert into t2(id, name,addr,email,phone,age,idc) values('@ksuid', '@姓名', '@地址', '@邮箱', '@手机', '@random_int(15-95)', '@身份证');
2023/11/02 10:43:54 SQL: insert into t2(id, name, addr, email, phone, age, idc) values (:1, :2, :3, :4, :5, :6, :7) ::: Args: ["2XbLrxPpXcdbIbyl...","雍璕碋","内蒙古自治\ufffd...","odldrzuz@lmtwd.s...","14565225308",86,"4181572009021599..."]
+--------------+--------------+--------------+
| lastInsertId | rowsAffected | cost         |
+--------------+--------------+--------------+
|            0 |            1 | 1.193626675s |
+--------------+--------------+--------------+
> select * from t2;
+-----------------------------+--------+--------------------------------------------------------+----------------------+-------------+-----+--------------------+
| ID                          | NAME   | ADDR                                                   | EMAIL                | PHONE       | AGE | IDC                |
+-----------------------------+--------+--------------------------------------------------------+----------------------+-------------+-----+--------------------+
| 2XbLrxPpXcdbIbyl005fBZqZure | 雍璕碋 | 内蒙古自治区乌兰察布市嗪貶路5924号黢蹵小区11单元1002室 | odldrzuz@lmtwd.space | 14565225308 | 86  | 418157200902159939 |
+-----------------------------+--------+--------------------------------------------------------+----------------------+-------------+-----+--------------------+
```

## 备份 vagrant 

```sh
➜  oracle12c-vagrant git:(master) ✗ vagrant halt
==> default: Attempting graceful shutdown of VM...
➜  oracle12c-vagrant git:(master) ✗ vagrant package --output oracle12.2.box 
➜  oracle12c-vagrant git:(master) ✗ rm /Volumes/e1t/soft/oracle12.2.box
➜  oracle12c-vagrant git:(master) ✗ vagrant package --output /Volumes/e1t/soft/oracle12.2.box
==> default: Clearing any previously set forwarded ports...
==> default: Exporting VM...
==> default: Compressing package to: /Volumes/e1t/soft/oracle12.2.box
➜  oracle12c-vagrant git:(master) ✗ ls -hl /Volumes/e1t/soft/oracle12.2.box
-rw-r--r--  1 bingoo  staff   5.8G 11  2 12:32 /Volumes/e1t/soft/oracle12.2.box
```

gzip 压缩没有效果

## 虚拟机信息

```sh
➜  oracle12c-vagrant git:(master) ✗ vagrant ssh
Last login: Thu Nov  2 02:36:12 2023 from 10.0.2.2
[vagrant@localhost ~]$ uname -a
Linux localhost.localdomain 5.4.17-2136.324.5.3.el7uek.x86_64 #2 SMP Tue Oct 10 12:44:19 PDT 2023 x86_64 x86_64 x86_64 GNU/Linux
[vagrant@localhost ~]$ cat /etc/*release
Oracle Linux Server release 7.9
NAME="Oracle Linux Server"
VERSION="7.9"
ID="ol"
ID_LIKE="fedora"
VARIANT="Server"
VARIANT_ID="server"
VERSION_ID="7.9"
PRETTY_NAME="Oracle Linux Server 7.9"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:oracle:linux:7:9:server"
HOME_URL="https://linux.oracle.com/"
BUG_REPORT_URL="https://github.com/oracle/oracle-linux"

ORACLE_BUGZILLA_PRODUCT="Oracle Linux 7"
ORACLE_BUGZILLA_PRODUCT_VERSION=7.9
ORACLE_SUPPORT_PRODUCT="Oracle Linux"
ORACLE_SUPPORT_PRODUCT_VERSION=7.9
Red Hat Enterprise Linux Server release 7.9 (Maipo)
Oracle Linux Server release 7.9
```