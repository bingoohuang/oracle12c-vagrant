
# Oracle 19c standalone（单机ASM）静默安装

[原文](https://www.modb.pro/db/366311?az1229)

记录一下Oracle 19c standalone（单机ASM）静默安装的流程，方便遇到这个场景的时候可以快速部署上。以下配置经过测试环境多次试错得出，如果有相同环境，改改ip主机名磁盘组应该就能直接用。

## 一、环境

CentOS 7.7 内存8G
磁盘组：DG_REDO 1Gx3 \ DG_DATA 10Gx2

## 二、配置

### 1.hosts文件

```sh
# 替换ip 主机名
echo "192.168.10.8 xkdg" >> /etc/hosts
cat /etc/hosts
```

### 2.语言环境

```sh
# root
echo "export LANG=en_US.UTF8" >> ~/.bash_profile
cat ~/.bash_profile
source .bash_profile
```

### 3.创建用户、组和目录

```sh
# 组
/usr/sbin/groupadd -g 60001 oinstall
/usr/sbin/groupadd -g 60002 dba
/usr/sbin/groupadd -g 60003 oper
/usr/sbin/groupadd -g 60004 backupdba
/usr/sbin/groupadd -g 60005 dgdba
/usr/sbin/groupadd -g 60006 kmdba
/usr/sbin/groupadd -g 60007 asmdba
/usr/sbin/groupadd -g 60008 asmoper
/usr/sbin/groupadd -g 60009 asmadmin

# 用户
useradd -u 61001 -g oinstall -G dba,asmdba,backupdba,dgdba,kmdba,oper oracle
useradd -u 61002 -g oinstall -G asmadmin,asmdba,asmoper,dba grid

# 设密码
passwd grid
passwd oracle

# 建目录
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/grid
mkdir -p /u01/app/oracle
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
mkdir -p /u01/app/oraInventory
chown -R grid:oinstall /u01
chmod -R 775 /u01
chown -R oracle:oinstall /u01/app/oracle
chown -R grid:oinstall /u01/app/oraInventory
```

4.yum源
--虚拟机环境记得先把盘挂上，再mount
mount /dev/sr0 /mnt

cd /etc/yum.repos.d
rm -rf *

cat <<EOF>> mnt.repo
[mnt]
name=mnt
baseurl=file:///mnt
gpgcheck=0
enabled=1
EOF

yum clean all
yum makecache
5.rpm包
下面应该是全了，如果在precheck环节检查出还有缺的，再单个装吧

yum -y install bc gcc gcc-c++ binutils compat-libcap1 compat-libstdc++ dtrace-modules dtrace-modules-headers dtrace-modules-provider-headers dtrace-utils elfutils-libelf elfutils-libelf-devel fontconfig-devel glibc glibc-devel ksh libaio libaio-devel libdtrace-ctf-devel libX11 libXau libXi libXtst libXrender libXrender-devel libgcc librdmacm-devel libstdc++ libstdc++-devel libxcb make smartmontools sysstat 

yum -y install kmod*
yum -y install ksh*
yum -y install libaio*
yum -y install compat*

注意：compat-libstdc++-33-3.2.3-69.el6.x86_64.rpm得单独找来装上

6.修改资源参数

--加到最后，单位KB
vi /etc/security/limits.conf

#SETTING for ORACLE
grid    soft  nproc   16384
grid    hard  nproc   16384
grid    soft  nofile  65536
grid    hard  nofile  65536
grid    soft  stack   32768
grid    hard  stack   32768
oracle  soft  nproc   16384
oracle  hard  nproc   16384
oracle  soft  nofile  65536
oracle  hard  nofile  65536
oracle  soft  stack   32768
oracle  hard  stack   32768
oracle  hard  memlock unlimited
oracle  soft  memlock unlimited

--控制给用户分配的资源
echo "session required pam_limits.so" >> /etc/pam.d/login
cat /etc/pam.d/login
7.修改内核参数
vi /etc/sysctl.conf

# Oracle install config
fs.aio-max-nr = 1048576
fs.file-max = 6815744 
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
kernel.panic_on_oops = 1
kernel.shmmax = 5033164800
kernel.shmall = 1228800
kernel.shmmni = 4096 
##Huge page
vm.nr_hugepages = 2500

--生效
sysctl -p
8.关闭透明大页
cat /sys/kernel/mm/transparent_hugepage/defrag
[always] madvise never
cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never

vi /etc/rc.d/rc.local

if test -f /sys/kernel/mm/transparent_hugepage/enabled;then
echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag;then
echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi

chmod +x /etc/rc.d/rc.local
9.关闭numa
vi /etc/default/grub

在GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet numa=off" 这一行加上numa=off

--重新编译
grub2-mkconfig -o /etc/grub2.cfg
10.共享内存段
vi /etc/fstab
none    /dev/shm       tmpfs    defaults,size=6G        0  0
--生效
mount -o remount    /dev/shm

注意：这里记录一下，配置完df -h能看到确实生效了，但是安装前检查还是有这个提示，得空再瞅瞅
11.禁用selinux，关闭防火墙
--禁用selinux
setenforce 0
vi /etc/selinux/config
--更改以下内容为disabled，重启操作系统才生效
SELINUX=disabled

--关闭防火墙：
systemctl stop firewalld
--禁止开机启动：
systemctl disable firewalld.service
12.配置环境变量
--grid
su - grid
vi ~/.bash_profile

export ORACLE_BASE=/u01/app/grid
export ORACLE_HOME=/u01/app/19.0.0/grid
export ORACLE_SID=+ASM
export LANG=en_US.UTF8
export NLS_DATE_FORMAT="yyyy-mm-dd hh24:mi:ss"
export PATH=.:${PATH}:$HOME/bin:$ORACLE_HOME/bin:$ORACLE_HOME/OPatch
export PATH=${PATH}:/usr/bin:/bin:/usr/bin/X11:/usr/local/bin
export PATH=${PATH}:$ORACLE_BASE/common/oracle/bin
export ORACLE_TERM=xterm
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:$ORACLE_HOME/oracm/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib
export CLASSPATH=$ORACLE_HOME/JRE
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/jlib
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/rdbms/jlib
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/network/jlib
export THREADS_FLAG=native
export TEMP=/tmp
export TMPDIR=/tmp
alias s='sqlplus / as sysdba'
umask 022
export TMOUT=0

--oracle 改sid、uniquename
su - oracle
vi ~/.bash_profile

export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.0.0/dbhome_1
export ORACLE_SID=xkdg
export ORACLE_UNQNAME=xkdg
export LANG=en_US.UTF-8
export NLS_LANG=american_america.UTF8
export NLS_DATE_FORMAT="yyyy-mm-dd hh24:mi:ss"
export PATH=.:${PATH}:$HOME/bin:$ORACLE_HOME/bin:$ORACLE_HOME/OPatch
export PATH=${PATH}:/usr/bin:/bin:/usr/bin/X11:/usr/local/bin
export PATH=${PATH}:$ORACLE_BASE/common/oracle/bin:/home/oracle/run
export ORACLE_TERM=xterm
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:$ORACLE_HOME/oracm/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib
export CLASSPATH=$ORACLE_HOME/JRE
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/jlib
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/rdbms/jlib
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/network/jlib
export THREADS_FLAG=native
export TEMP=/tmp
export TMPDIR=/tmp
export GI_HOME=/u01/app/19.0.0/grid
export PATH=${PATH}:$GI_HOME/bin
export ORA_NLS10=$GI_HOME/nls/data
umask 022
export TMOUT=0
13.虚拟机添加磁盘
VMware关虚拟机添加磁盘
DG_REDO 1Gx3
DG_DATA 10Gx2

加完记得修改虚拟机存放文件夹下的.vmx文件，添加
disk.EnableUUID = “TRUE”

14.UDEV配置
好用来源：https://www.modb.pro/issue/2279

1. 生成规则文件

 touch /etc/udev/rules.d/99-oracle-asmdevices.rules
 或者
 touch /usr/lib/udev/rules.d/99-oracle-asmdevices.rules

2. 生成规则
没有对sdb进行分区,执行如下shell脚本，
for i in b c d e f;
do
 echo "KERNEL==\"sd*\", SUBSYSTEM==\"block\", PROGRAM==\"/lib/udev/scsi_id --whitelisted --replace-whitespace --device=/dev/\$name\", RESULT==\"`/lib/udev/scsi_id --whitelisted --replace-whitespace --device=/dev/sd$i`\", SYMLINK+=\"asm-disk$i\", OWNER=\"grid\", GROUP=\"asmadmin\", MODE=\"0660\""      
done

3. 将结果复制到 99-oracle-asmdevices.rules 

将第二步的输出粘贴入 99-oracle-asmdevices.rules 这个文件

KERNEL=="sd*", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id --whitelisted --replace-whitespace --device=/dev/$name", RESULT=="36000c2948ef9d9e4a7937bfc65888bc8", NAME="asm-diskb", OWNER="grid", GROUP="asmadmin", MODE="0660"

执行
/sbin/partprobe /dev/sdb
/sbin/partprobe /dev/sdc
/sbin/partprobe /dev/sdd
/sbin/partprobe /dev/sde
/sbin/partprobe /dev/sdf

4. 用udevadm进行测试，注意udevadm命令不接受/dev/sdc这样的挂载设备名，必须是使用/sys/block/sdb这样的原始设备名。

udevadm test /sys/block/sdb
udevadm test /sys/block/sdc
udevadm test /sys/block/sdd
udevadm test /sys/block/sde
udevadm test /sys/block/sdf
udevadm info --query=all --path=/sys/block/sdb
udevadm info --query=all --path=/sys/block/sdc
udevadm info --query=all --path=/sys/block/sdd
udevadm info --query=all --path=/sys/block/sde
udevadm info --query=all --path=/sys/block/sdf
udevadm info --query=all --name=asm-diskb
udevadm info --query=all --name=asm-diskc
udevadm info --query=all --name=asm-diskd
udevadm info --query=all --name=asm-diske
udevadm info --query=all --name=asm-diskf

5. 启动udev
 /usr/sbin/udevadm control --reload-rules
 systemctl status systemd-udevd.service
 systemctl enable systemd-udevd.service

6. 检查设备是否正确绑定

# ls -l /dev/asm* /dev/sdb
lrwxrwxrwx 1 root root         3 Nov 29 18:17 /dev/asm-diskb -> sdb
brw-rw---- 1 grid asmadmin 8, 16 Nov 29 18:17 /dev/sdb
三、Grid安装
安装包：LINUX.X64_193000_grid_home.zip
上传路径： /u01/app/19.0.0/grid

--直接切到grid解压缩
su - grid
cd /u01/app/19.0.0/grid 
unzip LINUX.X64_193000_grid_home.zip

--解压完装个包
cd /u01/app/19.0.0/grid/cv/rpm
rpm -ivh cvuqdisk-1.0.10-1.rpm
1. 编辑响应文件
模板在 /u01/app/19.0.0/grid/inventory/response/grid_install.rsp
想了解参数具体用途的可以看模板里的说明

--touch gi.rsp
oracle.install.responseFileVersion=/oracle/install/rspfmt_crsinstall_response_schema_v19.0.0
INVENTORY_LOCATION=/u01/app/oraInventory
oracle.install.option=HA_CONFIG
ORACLE_BASE=/u01/app/grid
oracle.install.asm.OSDBA=asmdba
oracle.install.asm.OSOPER=asmoper
oracle.install.asm.OSASM=asmadmin
oracle.install.crs.config.scanType=LOCAL_SCAN
oracle.install.crs.config.ClusterConfiguration=STANDALONE
oracle.install.crs.config.configureAsExtendedCluster=false
oracle.install.crs.config.gpnp.configureGNS=false
oracle.install.crs.config.autoConfigureClusterNodeVIP=false
oracle.install.crs.config.gpnp.gnsOption=CREATE_NEW_GNS
oracle.install.crs.configureGIMR=false
oracle.install.asm.configureGIMRDataDG=false             
oracle.install.crs.config.useIPMI=false
oracle.install.asm.SYSASMPassword=Password123!
oracle.install.asm.diskGroup.name=DG_REDO
oracle.install.asm.diskGroup.redundancy=EXTERNAL
oracle.install.asm.diskGroup.AUSize=4
oracle.install.asm.diskGroup.disks=/dev/asm-diskb,/dev/asm-diskc,/dev/asm-diskd
oracle.install.asm.diskGroup.diskDiscoveryString=/dev/asm*
oracle.install.asm.monitorPassword=Password123!
oracle.install.asm.configureAFD=false
oracle.install.crs.configureRHPS=false
oracle.install.crs.config.ignoreDownNodes=false               
oracle.install.config.managementOption=NONE
oracle.install.config.omsPort=0
oracle.install.crs.rootconfig.executeRootScript=false

注意：oracle.install.asm.configureAFD=false 因为我的环境是CentOS所以设置的false，生产一般是RHEL得设为true
2. 安装前precheck
--有问题按提示调整
/u01/app/19.0.0/grid/runcluvfy.sh stage -pre crsinst -n xkdg -fixup -verbose
3. 安装过程
--安装脚本
$ORACLE_HOME/gridSetup.sh -silent -responseFile /u01/app/19.0.0/grid/inventory/response/gi.rsp

--执行结果，作为参考，记得手工执行俩脚本
[grid@xkdg response]$ $ORACLE_HOME/gridSetup.sh -silent -responseFile /u01/app/19.0.0/grid/inventory/response/gi.rsp
Launching Oracle Grid Infrastructure Setup Wizard...

[WARNING] [INS-32047] The location (/u01/app/oraInventory) specified for the central inventory is not empty.
   ACTION: It is recommended to provide an empty location for the inventory.
The response file for this session can be found at:
 /u01/app/19.0.0/grid/install/response/grid_2022-03-09_10-31-11AM.rsp

You can find the log of this install session at:
 /tmp/GridSetupActions2022-03-09_10-31-11AM/gridSetupActions2022-03-09_10-31-11AM.log

As a root user, execute the following script(s):
        1. /u01/app/oraInventory/orainstRoot.sh
        2. /u01/app/19.0.0/grid/root.sh

Execute /u01/app/19.0.0/grid/root.sh on the following nodes:
[xkdg]

Successfully Setup Software.
As install user, execute the following command to complete the configuration.
        /u01/app/19.0.0/grid/gridSetup.sh -executeConfigTools -responseFile /u01/app/19.0.0/grid/inventory/response/gi.rsp [-silent]


Moved the install session logs to:
 /u01/app/oraInventory/logs/GridSetupActions2022-03-09_10-31-11AM
[grid@xkdg response]$ exit
logout
[root@xkdg cdrom]# /u01/app/oraInventory/orainstRoot.sh
Changing permissions of /u01/app/oraInventory.
Adding read,write permissions for group.
Removing read,write,execute permissions for world.

Changing groupname of /u01/app/oraInventory to oinstall.
The execution of the script is complete.
[root@xkdg cdrom]# /u01/app/19.0.0/grid/root.sh
Check /u01/app/19.0.0/grid/install/root_xkdg_2022-03-09_10-33-58-938892079.log for the output of root script
[root@xkdg cdrom]#
4. 配置
--命令
/u01/app/19.0.0/grid/gridSetup.sh -executeConfigTools -responseFile /u01/app/19.0.0/grid/inventory/response/gi.rsp -silent

--执行结果
[grid@xkdg ~]$ /u01/app/19.0.0/grid/gridSetup.sh -executeConfigTools -responseFile /u01/app/19.0.0/grid/inventory/response/gi.rsp -silent
Launching Oracle Grid Infrastructure Setup Wizard...

You can find the logs of this session at:
/u01/app/oraInventory/logs/GridSetupActions2022-03-09_10-30-05PM

You can find the log of this install session at:
 /u01/app/oraInventory/logs/UpdateNodeList2022-03-09_10-30-05PM.log
Successfully Configured Software.
进行到这里可以 crsctl stat res -t 看看状态了

四、创建DG_DATA磁盘组
su - grid
sqlplus / as sysasm
create diskgroup DG_DATA EXTERNAL redundancy disk '/dev/asm-diske','/dev/asm-diskf' attribute 'compatible.rdbms'='19.0','compatible.asm'='19.0','au_size'='4M';
五、安装oracle软件
安装包：LINUX.X64_193000_db_home.zip
上传路径：/u01/app/oracle/product/19.0.0/dbhome_1
模板路径：/u01/app/oracle/product/19.0.0/dbhome_1/install/response/db_install.rsp

su - oracle
cd $ORACLE_HOME
unzip LINUX.X64_193000_db_home.zip
1. 编辑响应文件
--cd /u01/app/oracle/product/19.0.0/dbhome_1/install/response/
--touch db.rsp
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v19.0.0
oracle.install.option=INSTALL_DB_SWONLY
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.OSDBA_GROUP=dba
oracle.install.db.OSOPER_GROUP=dba
oracle.install.db.OSBACKUPDBA_GROUP=dba
oracle.install.db.OSDGDBA_GROUP=dba
oracle.install.db.OSKMDBA_GROUP=dba
oracle.install.db.OSRACDBA_GROUP=dba
oracle.install.db.rootconfig.executeRootScript=false
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE
oracle.install.db.config.starterdb.memoryOption=false
oracle.install.db.config.starterdb.installExampleSchemas=false
oracle.install.db.config.starterdb.managementOption=DEFAULT
oracle.install.db.config.starterdb.enableRecovery=false
2. 安装
--命令
cd $ORACLE_HOME
./runInstaller -silent -responseFile /u01/app/oracle/product/19.0.0/dbhome_1/install/response/db.rsp -skipPrereqs

--执行结果，作为参考，记得执行root.sh
[oracle@xkdg dbhome_1]$ ./runInstaller -silent -responseFile /u01/app/oracle/product/19.0.0/dbhome_1/install/response/db.rsp -skipPrereqs
Launching Oracle Database Setup Wizard...

The response file for this session can be found at:
 /u01/app/oracle/product/19.0.0/dbhome_1/install/response/db_2022-03-09_11-46-03PM.rsp

You can find the log of this install session at:
 /u01/app/oraInventory/logs/InstallActions2022-03-09_11-46-03PM/installActions2022-03-09_11-46-03PM.log

As a root user, execute the following script(s):
        1. /u01/app/oracle/product/19.0.0/dbhome_1/root.sh

Execute /u01/app/oracle/product/19.0.0/dbhome_1/root.sh on the following nodes:
[xkdg]


Successfully Setup Software.
[oracle@xkdg dbhome_1]$ exit
logout
[root@xkdg tmp]# /u01/app/oracle/product/19.0.0/dbhome_1/root.sh
Check /u01/app/oracle/product/19.0.0/dbhome_1/install/root_xkdg_2022-03-09_23-47-30-636609348.log for the output of root script

## 六、DBCA建库

模板 /u01/app/oracle/product/19.0.0/dbhome_1/assistants/dbca/dbca.rsp


1. 响应文件

cdb模式记得 createAsContainerDatabase=true

```ini
responseFileVersion=/oracle/assistants/rspfmt_dbca_response_schema_v19.0.0
gdbName=xkdg
sid=xkdg
databaseConfigType=SI
policyManaged=FALSE
createServerPool=FALSE
serverPoolName=
createAsContainerDatabase=true
numberOfPDBs=1
pdbName=test01
useLocalUndoForPDBs=true
pdbAdminPassword=Password123!
templateName=/u01/app/oracle/product/19.0.0/dbhome_1/assistants/dbca/templates/General_Purpose.dbc
sysPassword=Password123!
systemPassword= Password123!
runCVUChecks=true
dbsnmpPassword=Password123!
dvConfiguration=False
olsConfiguration=False
datafileJarLocation=/u01/app/oracle/product/19.0.0/dbhome_1/assistants/dbca/templates/
datafileDestination=DG_DATA
storageType=ASM
diskGroupName=DG_DATA
asmsnmpPassword=Password123!
recoveryGroupName=DG_DATA
characterSet=UTF8
nationalCharacterSet=AL16UTF16
registerWithDirService=FALSE
listeners=LISTENER
sampleSchema=FALSE
memoryPercentage=50
databaseType=MULTIPURPOSE
automaticMemoryManagement=FALSE
dbsnmpPassword=Password123!
```

2.执行

```sh
## 命令
dbca -silent -createDatabase -responseFile /u01/app/oracle/product/19.0.0/dbhome_1/assistants/dbca/dbca1.rsp

## 执行结果
[oracle@xkdg dbca]$ dbca -silent -createDatabase -responseFile /u01/app/oracle/product/19.0.0/dbhome_1/assistants/dbca/dbca1.rsp
Prepare for db operation
7% complete
Registering database with Oracle Restart
11% complete
Copying database files
33% complete
Creating and starting Oracle instance
35% complete
38% complete
42% complete
45% complete
48% complete
Completing Database Creation
53% complete
55% complete
56% complete
Creating Pluggable Databases
60% complete
78% complete
Executing Post Configuration Actions
100% complete
Database creation complete. For details check the logfiles at:
 /u01/app/oracle/cfgtoollogs/dbca/xkdg.
Database Information:
Global Database Name:xkdg
System Identifier(SID):xkdg
Look at the log file "/u01/app/oracle/cfgtoollogs/dbca/xkdg/xkdg.log" for further details.
[oracle@xkdg dbca]$
```

安装完成

## 七、小结

后续打补丁、刷参数、改redo等等等。
