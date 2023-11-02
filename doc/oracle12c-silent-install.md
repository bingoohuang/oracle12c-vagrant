# oracle12c静默安装（已验证）

https://blog.51cto.com/u_11639179/5418640

## 环境

    Centons7（内存2g，处理器：2，硬盘：40g）,jdk1.8,
    # 最小化安装系统，补充命令：
        a、更新系统： yum install -y update
        b、安装net-tools： yum install -y net-tools
        c、安装vim： yum install -y vim

1)下载 linuxx64_12201_database.zip

https://www.oracle.com/database/technologies/oracle12c-linux-12201-downloads.html#license-lightbox

2）修改IP和主机名，要设置一下HOSTS配置文件，不然启动oracle监听和实例会比较慢

vim /etc/hosts
ip + 主机名

3）检查oracle所需包 （以root身份操作）

```sh
rpm -q binutils compat-libcap1 compat-libstdc++-33 compat-libstdc++-33*.i686 glibc glibc*.i686 glibc-devel glibc-devel*.i686 ksh libaio libaio*.i686 libaio-devel libaio-devel*.i686 libX11 libX11*.i686 libXau libXau*.i686 libXi libXi*.i686 libXtst libXtst*.i686 libgcc libgcc*.i686 libstdc++ libstdc++*.i686 libstdc++-devel libstdc++-devel*.i686  libxcb libxcb*.i686 make nfs-utils net-tools smartmontools sysstat unixODBC unixODBC-devel gcc gcc-c++ libXext libXext*.i686 zlib-devel zlib-devel*.i686 unzip
```

4）安装oracle依赖包（以root身份操作）

```sh
yum install -y binutils compat-libcap1 compat-libstdc++-33 compat-libstdc++-33*.i686 glibc glibc*.i686 glibc-devel glibc-devel*.i686 ksh libaio libaio*.i686 libaio-devel libaio-devel*.i686 libX11 libX11*.i686 libXau libXau*.i686 libXi libXi*.i686 libXtst libXtst*.i686 libgcc libgcc*.i686 libstdc++ libstdc++*.i686 libstdc++-devel libstdc++-devel*.i686  libxcb libxcb*.i686 make nfs-utils net-tools smartmontools sysstat unixODBC unixODBC-devel gcc gcc-c++ libXext libXext*.i686 zlib-devel zlib-devel*.i686 unzip
```

##  也可以分段执行，避免遗漏

```sh
yum install -y compat-libcap1 compat-libstdc++-33 compat-libstdc++-33*.i686
yum install -y glibc*.i686 glibc-devel glibc-devel*.i686
yum install -y ksh libaio*.i686 libaio-devel libaio-devel*.i686
yum install -y libX11 libX11*.i686 libXau libXau*.i686 libXi libXi*.i686
yum install -y libXtst libXtst*.i686 libgcc*.i686 libstdc++*.i686 libstdc++-devel libstdc++-devel*.i686
yum install -y libxcb libxcb*.i686 make nfs-utils net-tools smartmontools sysstat unixODBC unixODBC-devel
yum install -y gcc gcc-c++ libXext libXext*.i686 zlib-devel zlib-devel*.i686 unzip
```

5）修改内核参数（参考的资料有这么一段，考虑到这个是系统性能优化的设置，所以没有做这一步）

```sh
[root@Oracle ~]# vim /etc/sysctl.conf

#修改或添加以下内容
fs.aio-max-nr = 1048576
fs.file-max = 6815744               // 设置最大打开文件数
kernel.shmall = 16777216            // 共享内存的总量，8G内存设置：2097152*4k/1024/1024
kernel.shmmax = 34359738360         // 最大共享内存的段大小，G换算成k计算
kernel.shmmni = 4096                // 整个系统共享内存端的最大数
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500        // 可使用的IPv4端口范围
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
#
[root@Oracle ~]# sysctl -p

# 检查并生效
[root@Oracle ~]# sysctl -a
```

6）创建用户组和用户（以root身份操作）

```sh
[root@Oracle ~]# groupadd oinstall
[root@Oracle ~]# groupadd dba
[root@Oracle ~]# groupadd oper
[root@Oracle ~]# useradd -g oinstall -G dba,oper oracle
#修改用户密码
[root@Oracle ~]# passwd oracle
或
[root@Oracle ~]# echo "123456" | passwd --stdin oracle
```sh

7）创建相关目录

```sh
[root@Oracle ~]# mkdir /u01
[root@Oracle ~]# mkdir -p /u01/app/oracle                 //oracle数据库安装目录
[root@Oracle ~]# mkdir -p /u01/app/oraInventory           //oracle数据库配置文件目录
[root@Oracle ~]# mkdir -p /u01/app/oracle/oradata         //存放数据库的数据目录
[root@Oracle ~]# mkdir -p /u01/app/oracle/oradata_back    //存放数据库备份文件
[root@Oracle ~]# chmod -R 775 /u01/app
[root@Oracle ~]# chown -R oracle:oinstall /u01            //设置目录所有者为oinstall用户组的oracle用户
```

8）修改 etc/profile

```sh
if [ $USER = "oracle" ]; then
   if [ $SHELL = "/bin/ksh" ]; then
       ulimit -p 16384
       ulimit -n 65536a
   else
       ulimit -u 16384 -n 65536
   fi
fi
```

9）修改/home/oracle/.bash_profile

```sh
#oracle数据库安装目录
ORACLE_BASE=/u01/app/oracle
#oracle数据库路径
ORACLE_HOME=$ORACLE_BASE/product/12.2.0/db_1
#oracle启动数据库实例名
ORACLE_SID=orcl
#添加系统环境变量
PATH=$PATH:$HOME/bin:$ORACLE_HOME/bin
#添加系统环境变量
LD_LIBRARY_PATH=$ORACLE_HOME/lib:/usr/lib
# 该部分重要，后续错误一般是该步骤造成
export ORACLE_BASE ORACLE_HOME ORACLE_SID PATH LD_LIBRARY_PATH

# 使配置生效
[root@Oracle ~]# source /home/oracle/.bash_profile
```

 10）本地安装jdk，并上传：linuxx64_12201_database.zip 文件

```sh
[root@Oracle ~]# java -version                     # 查看安装版本
[root@Oracle ~]# yum remove openjdk                # 如果是系统自带的openjdk，则卸载
[root@Oracle ~]# rpm -ivh jdk-8u191-linux-x64.rpm  # 安装自己下载的JDK
```

11）解压oracle安装包，解压后路径：/u01/database

```sh
[root@Oracle ~]# unzip linuxx64_12201_database.zip -d /u01        # 没有unzip命令，则先安装unzip
```

12）修改应答文件，静默安装配置文件路径：/u01/database/response/db_install.rsp

```sh
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v12.2.0
# 30行 安装类型,只装数据库软件
oracle.install.option=INSTALL_DB_SWONLY
# 35行 用户组
UNIX_GROUP_NAME=oinstall
# 42行 INVENTORY目录（不填就是默认值）
INVENTORY_LOCATION=/u01/app/oraInventory
# 46行 oracle目录
ORACLE_HOME=/u01/app/oracle/product/12.2.0/db_1
# 51行 oracle基本目录
ORACLE_BASE=/u01/app/oracle
# 63行 oracle版本
oracle.install.db.InstallEdition=EE
# 80行
oracle.install.db.OSDBA_GROUP=dba
# 86行
oracle.install.db.OSOPER_GROUP=oper
# 91行
oracle.install.db.OSBACKUPDBA_GROUP=dba
# 96行
oracle.install.db.OSDGDBA_GROUP=dba
# 101行
oracle.install.db.OSKMDBA_GROUP=dba
# 106行
oracle.install.db.OSRACDBA_GROUP=dba
# 180行 数据库类型
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE
# 185行
oracle.install.db.config.starterdb.globalDBName=orcl
# 190行
oracle.install.db.config.starterdb.SID=orcl
# 216行
oracle.install.db.config.starterdb.characterSet=AL32UTF8
# 384行
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
# 398行 设置安全更新（貌似是有bug，这个一定要选true，否则会无限提醒邮件地址有问题，终止安装。PS：不管地址对不对）
DECLINE_SECURITY_UPDATES=true
```

13）安装Oracle数据库软件（以oracle用户身份操作，执行需要几分钟）

```sh
[oracle@vs database]$ ./runInstaller -force -silent -noconfig -ignorePrereq -ignoreSysPreReqs -responseFile /u01/database/response/db_install.rsp
# 执行过程中会有日志文件记录提示，可以tail -f 日志文件查看
```

eg: 可能出现的错误：/oui/lib/linux64/liboraInstaller.so: libnsl.so.1: cannot open shared object file: No such file or directory
原因没有安装libnsl-64位的包： dnf install libnsl.x86_64

14）安装成功后会提示如下图命令，需要切换到root身份执行

以 root 用户的身份执行以下脚本:
  1. /u01/app/oraInventory/orainstRoot.sh
  2. /u01/app/oracle/product/12.2.0/db_1/root.sh

15）配置监听，静默安装配置文件路径：/u01/database/response/netca.rsp

```sh
[oracle@vs ~]$ netca -silent -responsefile /u01/database/response/netca.rsp
# netca 是oracle net configuration assistance的简称，主要作用是配置监听程序、命名方法配置、本地net服务吗配置、目录使用配置

可以通过 netstat -tlnp 命令查看监听地址
netstat -tlnp
tcp  0   0 :::1521        :::*      LISTEN      5477/tnslsnr
```

16）修改dbca.rsp文件，静默安装配置文件路径：/u01/database/response/dbca.rsp

```sh
# 21行 不可更改
responseFileVersion=/oracle/assistants/rspfmt_dbca_response_schema_v12.2.0
# 32行 全局数据库名
gdbName=orcl
# 42行 系统标识符
sid=orcl
# 52行
databaseConfigType=SI
# 74行
policyManaged=false
# 88行
createServerPool=false
# 127行
force=false
# 163行
createAsContainerDatabase=true
# 172行
numberOfPDBs=1
# 182行
pdbName=orclpdb
# 192行
useLocalUndoForPDBs=true
# 203行 库密码
pdbAdminPassword=********
# 223行
templateName=/u01/app/oracle/product/12.2.0/db_1/assistants/dbca/templates/General_Purpose.dbc
# 233行 超级管理员密码
sysPassword=********
# 233行 管理员密码
systemPassword=********
# 273行
emExpressPort=5500
# 284行
runCVUChecks=false
# 313行
omsPort=0
# 341行
dvConfiguration=false
# 391行
olsConfiguration=false
# 401行
datafileJarLocation={ORACLE_HOME}/assistants/dbca/templates/
# 411行
datafileDestination={ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/
# 421行
recoveryAreaDestination={ORACLE_BASE}/fast_recovery_area/{DB_UNIQUE_NAME}
# 431行
storageType=FS
# 468行 字符集创建库之后不可更改
characterSet=AL32UTF8
# 478行
nationalCharacterSet=AL16UTF16
# 488行
registerWithDirService=false
# 526行
listeners=LISTENER
# 546行
variables=DB_UNIQUE_NAME=orcl,ORACLE_BASE=/u01/app/oracle,PDB_NAME=,DB_NAME=orcl,ORACLE_HOME=/u01/app/oracle/product/12.2.0.1/db_1,SID=orcl
# 555行
initParams=undo_tablespace=UNDOTBS1,memory_target=796MB,processes=300,db_recovery_file_dest_size=2780MB,nls_language=AMERICAN,dispatchers=(PROTOCOL=TCP) (SERVICE=orclXDB),db_recovery_file_dest={ORACLE_BASE}/fast_recovery_area/{DB_UNIQUE_NAME},db_block_size=8192BYTES,diagnostic_dest={ORACLE_BASE},audit_file_dest={ORACLE_BASE}/admin/{DB_UNIQUE_NAME}/adump,nls_territory=AMERICA,local_listener=LISTENER_orcl,compatible=12.2.0,control_files=("{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/control01.ctl", "{ORACLE_BASE}/fast_recovery_area/{DB_UNIQUE_NAME}/control02.ctl"),db_name=cdb1,audit_trail=db,remote_login_passwordfile=EXCLUSIVE,open_cursors=300
# 565行
sampleSchema=false
# 574行
memoryPercentage=40
# 584行
databaseType=MULTIPURPOSE
# 594行
automaticMemoryManagement=false
# 604行
totalMemory=0
```

17）创建数据库实例，使用 dbca 命令（dbca是oracle命令，如果提示命令找不到，检查环境变量，可能需要几分钟）

```sh
[oracle@oracle response]$ dbca -silent -createDatabase -responseFile  /u01/database/response/dbca.rsp
```

18）检查oracle进程状态

```sh
ps -ef | grep ora_ | grep -v grep
lsnrctl status
```

19）数据库实例的启动和关闭【只查看不，服务正常执行修改就行】

```sh
# 以 DBA 身份进入 sqlplus，查看数据库状态
[oracle@Oracle ~]$ sqlplus / as sysdba
SQL> select open_mode from v$database;             //查看数据库
SQL> select status from v$instance;                //查看数据库实例

# 以 DBA 身份进入 sqlplus，修改管理员用户密码
[oracle@Oracle ~]$ sqlplus / as sysdba
SQL> alter user sys identified by ********;        //改sys超级管理员密码
SQL> alter user system identified by ********;     //改system管理员密码

# 以 DBA 身份进入 sqlplus，启动数据库
[oracle@Oracle ~]$ sqlplus / as sysdba
SQL> startup

# 以 DBA 身份进入 sqlplus，关闭数据库
[oracle@Oracle ~]$ sqlplus / as sysdba
SQL> shutdown abort
或
SQL> shutdown immediate
```

20）修改oracle启动配置文件

完成oracle12c数据库的安装后，相关服务器会自动启用，但并不表示下次开机后oracle服务器仍然可用。下面将介绍oracle的基本服务组件，以及如何编写服务脚本来控制oracle数据库系统的自动运行。
根据上面的安装过程，oracle的数据库软件将安装在变量ORACLE_HOME所指向的位置。例如 /u01/app/oracle/product/12.2.0/db_1/ ，而各种服务器组件程序（也包括sqlplus命令）正是位于其中的bin子目录下。
Oracle数据库的基本服务组件如下所述：（注：oracle服务组件最好以oracle用户身份运行如：su - oracle）
lsnrctl：监听器程序，用来提供数据库访问，默认监听TCP 1521端口。
dbstart、dbshut：数据库控制程序，用来启动、停止数据库实例。
emctl:管理器控制工具，用来控制OEM平台的开启与关闭，OEM平台通过1158端口提供HTTPS访问，5520端口提供TCP访问。
为了方便执行oracle的服务组件程序，建议对所有用户的环境配置作进一步的优化调整、补充PATH路径、oracle终端类型等变量设置。除此以外，还应该修改/etc/oratab配置文件，以便运行dbstart时自动启用数据库实例。

```sh
# 修改oratab配置如下，这样就可以通过dbstart 启动实例，也可以通过dbshut关闭实例。
[oracle@Oracle ~]$ vim /etc/oratab

racl:/u01/app/oracle/product/12.2.0/db_1:Y  //把“N”改成“Y”

# 此时所有oracle的进程关闭，监听器也停止。
[oracle@Oracle ~]$ dbshut /u01/app/oracle/product/12.2.0/db_1/

# 启动监听器和实例。
[oracle@Oracle ~]$ dbstart /u01/app/oracle/product/12.2.0/db_1/

# 修改 dbstart 和 dbshut，如下 （修改后服务器重启才能正常使用）
# 修改 #ORACLE_HOME_LISTNER=$1 为 ORACLE_HOME_LISTNER=$ORACLE_HOME

# 在 root 用户下编辑 rc.local
# dbstart 默认将 oratab 中参数为 Y 的所有库启动
[root@Oracle ~]# vim /etc/rc.d/rc.local

# 添加如下命令到 rc.local
# 用oracle用户登录，运行dbstart启动数据库
# 添加，防止sqlplus 链接【oracle 12c 远程访问显示 ORA-12541:TNS:无监听程序】
su - oracle -lc "/u01/app/oracle/product/12.2.0/db_1/bin/dbshut"
su - oracle -lc "/u01/app/oracle/product/12.2.0/db_1/bin/dbstart"

# 设置执行权限，因为Oracle linux 7.x 默认rc.local是没有执行权限，需执行chmod自己增加
[root@Oracle ~]# chmod +x /etc/rc.d/rc.local

# 查看监听状态及数据库状态
[oracle@Oracle ~]$ lsnrctl status

# 启动监听
[oracle@Oracle ~]$ lsnrctl start

# 停止监听
[oracle@Oracle ~]$ lsnrctl stop
```

21）添加oracle 环境变量

```sh
export ORACLE_BASE=/u01/app/oracle;
export ORACLE_HOME=$ORACLE_BASE/product/12.2.0/db_1;
export ORACLE_SID=orcl;
export PATH=$ORACLE_HOME/bin:$PATH;
```

22）防火墙开放端口（以root身份操作，可以直接关掉，使用 iptables 防火墙）

```sh
# 开启端口
[root@vs ~]# firewall-cmd --zone=public --add-port=1521/tcp --permanent
# 重启防火墙 （一般我们在开放完新的端口后，需要重新启动防火墙）
[root@vs ~]# firewall-cmd --reload
```

 23）ORA-28040: 没有匹配的验证协议

```sh
在 $ORACLE_HOME/network/admin/sqlnet.ora
  加入如下:
  SQLNET.ALLOWED_LOGON_VERSION=8
```

24）用户创建

```sh
CDB: 公共用户创建需要带 c##
sqlplus / as sysdba
#创建用户
create user C##test identified by test;
create user C##stan identified by stan;
#删除用户
drop user C##test;
# 受权限
grant connect,resource,dba to c##test;
grant connect,resource,dba to c##stan;
# 撤销权限
revoke connect, resource from c##test;
```

25）监听问题： tnsnames.ora 修改文件中的loclahost为ip地址（不一定修改）

登录后复制
$ORACLE_HOME/product/12.2.0/db_1/network/admin/tnsnames.ora

```sql
select * from dba_users;
select * from all_users;
select * from user_users;
```
