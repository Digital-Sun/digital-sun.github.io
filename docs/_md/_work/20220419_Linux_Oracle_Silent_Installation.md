# 背景介绍
公司今年要新上WMS系统，计划阿里云上部署3台Linux服务器，用于安装Oracle数据库。
由于一些原因，数据库的部署并没有交予实施商，我接下了这个活。
一开始本意是打算梳理一下Linux下安装Oracle的文档，结果顺道尝试了使用ResponseFile进行slient安装。
也算是有新的收获。

---
# 前言
有一些历史遗留问题。
公司老的Oracle大部分是11.2.0.3，然后夹杂了很多Dblink。
为防止后续运维人员无意使用dblink连接高，低两个版本的数据库，带来[SCN BUG](https://mp.weixin.qq.com/s/ZPdLb3gDdz54GyfXtuSWpA)，
还是从SUPPORT上下载11.2.0.3 patchset Linux对应的安装程序
* p10404530_112030_Linux-x86-64_1of7.zip
* p10404530_112030_Linux-x86-64_2of7.zip

安装过程中主要参考的文档是官方Docs上的两篇文档：
- [Preinstallation Tasks](https://docs.oracle.com/cd/E11882_01/install.112/e47689/pre_install.htm#LADBI1085)
- [Installing and Configuring Oracle Database Using Response Files](https://docs.oracle.com/cd/E11882_01/install.112/e47689/app_nonint.htm#LADBI1341)

# 部署过程
## 获取服务器
基础架构的同事从阿里云上按规划购买了服务器，预制了CentOS系统，连接方式和Root登录密码给到了我。

## 防火墙关闭

拿到手，第一时间还是关闭了防火墙和selinux。
```shell
# CentOS7以下版本关闭防火墙和服务自启动
service iptables stop
chkconfig iptables off
# CentOS7及以上版本关闭防火墙和服务自启动
systemctl stop firewalld
systemctl disable firewalld
# 查看selinux状态，如果不是disabled需要修改设置。
getenforce
vi /etc/selinux/config
# 确保SELINUX=disabled
```

## 挂载准备

### 挂磁盘

首先，***df -h*** 查看了一下当前存储空间状态。
根目录下就36G可用空间，盲猜是估计同事没帮着挂载磁盘。（64位Linux的Oracle，系统要4.7G，数据文件要1.7G）
执行了一下命令
```shell
ls -l /dev/sd*
ls -l /dev/vd*
fdisk -l
```
果然**500G**的 ***/dev/vdb*** 没处理。
没啥好说的，分一个主分区就好了，然后制作文件格式，挂载跟目录新创建的 ***/u01*** 上。
```shell
fdisk /dev/sdb
# 输入n，add a new partition
# 一路回车，输入w写下500G分区
mkfs -t ext4  /dev/vdb1
mkdir /u01 
mount  /dev/vdb1 /u01
# 个人习惯，创建一个安装介质存放路径
mkdir /u01/Install_Media 
```
通常为了重启自动挂载，需要改一下 ***/etc/fstab*** ，不过还有一部分挂载工作需要做，所以可以一会儿一起设置。

### SWAP修改

按照Preinstallation Tasks的要求，需要结合机器内存检查一下SWAP大小：
```shell
grep MemTotal /proc/meminfo
# 更直观看一下大小
free -m
```
|RAM|SwapSpace|
|--|--|
|Between 1 GB and 2 GB|1.5 times the size of the RAM|
|Between 2 GB and 16 GB|Equal to the size of the RAM|
|More than 16 GB|16 GB|

我这台给了30G内存（微妙的少了2G），所以我需要做16G的SWAP。
```shell
dd if=/dev/zero of=/u01/swap1 bs=1M count=16384
mkswap /u01/swap1
swapon /u01/swap1
# 检查一下输出是不是你制作的大小
grep SwapTotal /proc/meminfo
```
当然这个也是要改 ***/etc/fstab*** 的

### tmpfs修改

按照Preinstallation Tasks的要求，如果要启用***AMM(Automatic Memory Management)*** ，得确保tmpfs大于 ***MEMORY_MAX_TARGET*** and ***MEMORY_TARGET*** 这两个参数设置的值，不然后面会启动报错。
我就偷懒直接改32G了。
```shell
# 查看改之前shared memory
df -h /dev/shm/
mount -t tmpfs shmfs -o size=32g /dev/shm
# 查看改之后shared memory
df -h /dev/shm/
```
现在，是时候修改一下 ***/etc/fstab*** 了。
添加下面这三行，注意挂载位置和被挂载位置的前后顺序。
```
/dev/vdb1                                 /u01                    ext4    defaults        0 0
/u01/swap1                                swap                    swap    defaults        0 0
shmfs                                     /dev/shm                tmpfs   defaults,size=32G 0 0
```

## 依赖包准备

按照Preinstallation Tasks的要求,找到自己版本编对应的依赖包。
我是 ***Oracle Linux 7 and Red Hat Enterprise Linux 7***
对应的依赖包有：
- binutils-2.23.52.0.1-12.el7.x86_64 
- compat-libcap1-1.10-3.el7.x86_64 
- compat-libstdc++-33-3.2.3-71.el7.i686
- compat-libstdc++-33-3.2.3-71.el7.x86_64
- gcc-4.8.2-3.el7.x86_64 
- gcc-c++-4.8.2-3.el7.x86_64 
- glibc-2.17-36.el7.i686 
- glibc-2.17-36.el7.x86_64 
- glibc-devel-2.17-36.el7.i686 
- glibc-devel-2.17-36.el7.x86_64 
- ksh
- libaio-0.3.109-9.el7.i686 
- libaio-0.3.109-9.el7.x86_64 
- libaio-devel-0.3.109-9.el7.i686 
- libaio-devel-0.3.109-9.el7.x86_64 
- libgcc-4.8.2-3.el7.i686 
- libgcc-4.8.2-3.el7.x86_64 
- libstdc++-4.8.2-3.el7.i686 
- libstdc++-4.8.2-3.el7.x86_64 
- libstdc++-devel-4.8.2-3.el7.i686 
- libstdc++-devel-4.8.2-3.el7.x86_64 
- libXi-1.7.2-1.el7.i686 
- libXi-1.7.2-1.el7.x86_64 
- libXtst-1.2.2-1.el7.i686 
- libXtst-1.2.2-1.el7.x86_64 
- make-3.82-19.el7.x86_64 
- sysstat-10.1.5-1.el7.x86_64 

老实的用yum安装完了依赖包

>yum install binutils compat-libcap1 compat-libstdc++ gcc gcc-c++ glibc glibc-devel ksh libaio libaio-devel libgcc libstdc++ libstdc++-devel libXi libXtst make sysstat unzip

## 关闭transparent hugepage
尽管THP的本意是为提升性能，但某些数据库厂商还是建议直接关闭THP(比如说O记)，否则可能导致性能下降，内存锁，甚至系统重启等问题。
Root用户登录。
```shell
# Red Hat Enterprise Linux kernels:
cat /sys/kernel/mm/redhat_transparent_hugepage/enabled
# Other kernels:
cat /sys/kernel/mm/transparent_hugepage/enabled
# 输出结果如下
The following is a sample output that shows Transparent HugePages are being used as the [always] flag is enabled.
[always] never
# 可以看到框在always上，需要调整到never
# 需要通过修改启动项去更改设置。结合自己的启动情况，我这里是grub2引导。
# 在启动文件 /boot/grub2/grub.cfg 里的启动项后面添加transparent_hugepage=never
# 在类似这种行的最后添加
linux16 /boot/vmlinuz-3.10.0-957.21.3.el7.x86_64 root=UUID=1114fe9e-2309-4580-b183-d778e6d97397 ro crashkernel=auto rhgb quiet idle=halt biosdevname=0 net.i
fnames=0 console=tty0 console=ttyS0,115200n8 noibrs nvme_core.io_timeout=4294967295 nvme_core.admin_timeout=4294967295 transparent_hugepage=never
```
到这里，可以重启试一试了。正好验证一下前面操作，检查磁盘是否正常挂载，大页是否修改了。

## 用户和目录准备

按照Preinstallation Tasks的要求 创建安装用户。

```shell
groupadd oinstall                                                                             
groupadd -g 502 dba                                                                           
groupadd -g 503 oper                                                                          
groupadd -g 504 asmadmin                                                                      
groupadd -g 505 asmoper                                                                       
groupadd -g 506 asmdba
useradd -u 501 -g oinstall -G dba,oper,asmdba oracle
passwd oracle # 好孩子不要用这种简单的密码
# 创建一个Inventory目录供后续安装使用，注意不要放在预定的Oracle Base中
mkdir /home/oracle/oraInventory
chown oracle:oinstall /home/oracle/oraInventory
# 检查一下oracle用户
id oracle
# 显示内容如下
uid=502(oracle) gid=1000(oinstall) groups=1000(oinstall),502(dba),503(oper),506(asmdba)
# 创建规划的软件目录
mkdir -p /u01/app/oracle
chown -R oracle:oinstall /u01/app/oracle
chmod -R 775 /u01/app/oracle
```

## 修改Resource Limits

按照Preinstallation Tasks的要求：
|Resource Shell Limit|Resource|Soft Limit|Hard Limit|
|-|-|-|-|
|Open file descriptors|nofile|at least 1024|at least 65536|
|Number of processes available to a single user|nproc|at least 2047|at least 16384|
|Size of the stack segment of the process|stack|at least 10240 KB|at least 10240 KB, and at most 32768 KB|
使用Oracle（安装用户）检查一下对应值是否满足要求。
```shell
$ ulimit -Sn
1024
$ ulimit -Hn
65536
$ ulimit -Su
2047
$ ulimit -Hu
16384
$ ulimit -Ss
10240
$ ulimit -Hs
32768
```
我的stack值不满足，于是修改了一下 ***/etc/security/limits.conf***，在文件末尾添加两行。
- oracle soft stack 10240
- oracle hard stack 32768

## 修改内核参数
按照Preinstallation Tasks的要求在 ***/etc/sysctl.conf*** 最后添加：
```
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 4294967295
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
```
运行 ***sysctl -p*** 重新装载一下

## 上传安装介质

把之前下载好的 1of7 和 2of7 按你喜欢的方法拷贝（我用的scp）到 ***/u01/Install_Media*** 下面，
两个压缩包unzip到一起。
文件目录如下
```shell
[root@WMS-ZJK-DB database]# ls -l
total 64
drwxr-xr-x 12 root root  4096 Sep 19  2011 doc
drwxr-xr-x  4 root root  4096 Sep 22  2011 install
-rwxr-xr-x  1 root root 28122 Sep 22  2011 readme.html
drwxr-xr-x  2 root root  4096 Sep 22  2011 response
drwxr-xr-x  2 root root  4096 Sep 22  2011 rpm
-rwxr-xr-x  1 root root  3226 Sep 22  2011 runInstaller
drwxr-xr-x  2 root root  4096 Sep 22  2011 sshsetup
drwxr-xr-x 14 root root  4096 Sep 22  2011 stage
-rwxr-xr-x  1 root root  5466 Aug 23  2011 welcome.html
```
>*这步当时做完已经20:00，这个时候我意识到，阿里云上的服务器，ping不通我笔记本，同事估计一时也处理不了,我没办法把图形界面x11转发到我屏幕。于是决定整个活儿，试试静默安装。*

## 静默安装

安装介质目录下有一个response文件夹。
```shell
[root@WMS-ZJK-DB response]# pwd
/u01/Install_Media/database/response
[root@WMS-ZJK-DB response]# ls -l
total 80
-rwxr-xr-x 1 root root 44533 Sep 22  2011 dbca.rsp
-rwxr-xr-x 1 root root 24992 Sep 22  2011 db_install.rsp
-rwxr-xr-x 1 root root  5871 Sep 22  2011 netca.rsp
```
三个rsp文件，安装使用**db_install.rsp**，创建数据库使用**dbca.rsp**，监听等配置使用**netca.rsp**

图像界面安装过的话，其实不难理解，相当于预先设置好相应的参数，O记直接调用这些进行安装，rsp中的说明也比较详细。

首先将这3个文件拷贝到一个其他目录。

O记建议修改文件权限为700，主要因为文件中涉及数据库账号密码，安装完后文件记得删除或移除处理。

---
到这里建议配置一下Oracle用户的环境变量,根据自己的情况修改内容，方便后面登录和执行命令。
```
export PATH
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1
export ORACLE_SID=<ORACLE_SID>
export PATH=$ORACLE_HOME/bin:$PATH
umask 022
```
---


那我按照以下三部进行：
- 仅安装数据库软件
- 配置数据库实例
- 配置监听

### 安装数据库软件

来看看db_install.rsp，很长，其实官方注释已经很明细了，我还是在这里用中文结合图形界面安装注释一些内容，所以文件里中文都是我后添加的，使用的时候注意一下去掉#。

简化版内容如下:
```shell
####################################################################
## Simple Response File for Install DB Software Only              ##
####################################################################
#------------------------------------------------------------------------------
# Value do not be changed.
#------------------------------------------------------------------------------
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v11_2_0
oracle.install.db.EEOptionsSelection=false
oracle.install.db.optionalComponents=oracle.rdbms.partitioning:11.2.0.3.0,oracle.oraolap:11.2.0.3.0,oracle.rdbms.dm:11.2.0.3.0,oracle.rdbms.dv:11.2.0.3.0,oracle.rdbms.lbac:11.2.0.3.0,oracle.rdbms.rat:11.2.0.3.0
#------------------------------------------------------------------------------
# Value can be changed.
#-------------------------------------------------------------------------------
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=<HOSTNAME>
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/home/oracle/oraInventory
SELECTED_LANGUAGES=en,zh_CN
ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=oinstall
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE
oracle.install.db.config.starterdb.password.ALL=<password>
DECLINE_SECURITY_UPDATES=true
```


<details>
<summary><font color='#ea6f5a'><b><i>单击展开 db_install.rsp 详细内容</i></b></font></summary>

```shell
[oracle@WMS-ZJK-DB response]$ cat db_install.rsp 
####################################################################
## Copyright(c) Oracle Corporation 1998,2011. All rights reserved.##
##                                                                ##
## Specify values for the variables listed below to customize     ##
## your installation.                                             ##
##                                                                ##
## Each variable is associated with a comment. The comment        ##
## can help to populate the variables with the appropriate        ##
## values.                                                        ##
##                                                                ##
## IMPORTANT NOTE: This file contains plain text passwords and    ##
## should be secured to have read permission only by oracle user  ##
## or db administrator who owns this installation.                ##
##                                                                ##
####################################################################

#------------------------------------------------------------------------------
# Do not change the following system generated value. 
#------------------------------------------------------------------------------
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v11_2_0

#------------------------------------------------------------------------------
# Specify the installation option.
# It can be one of the following:
# 1. INSTALL_DB_SWONLY
# 2. INSTALL_DB_AND_CONFIG
# 3. UPGRADE_DB
# Steve  : 仅安装软件的话，就填第一个
#-------------------------------------------------------------------------------
oracle.install.option=INSTALL_DB_SWONLY

#-------------------------------------------------------------------------------
# Specify the hostname of the system as set during the install. It can be used
# to force the installation to use an alternative hostname rather than using the
# first hostname found on the system. (e.g., for systems with multiple hostnames 
# and network interfaces)
#-------------------------------------------------------------------------------
ORACLE_HOSTNAME=<HOSTNAME>

#-------------------------------------------------------------------------------
# Specify the Unix group to be set for the inventory directory.  
#-------------------------------------------------------------------------------
UNIX_GROUP_NAME=oninstall

#-------------------------------------------------------------------------------
# Specify the location which holds the inventory files.
# This is an optional parameter if installing on
# Windows based Operating System.
# Steve  : 这个得设置一下，不然会报错，设置在Oracle Base之外
#-------------------------------------------------------------------------------
INVENTORY_LOCATION=/home/oracle/oraInventory

#-------------------------------------------------------------------------------
# Specify the languages in which the components will be installed.             
# 
# en   : English                  ja   : Japanese                  
# fr   : French                   ko   : Korean                    
# ar   : Arabic                   es   : Latin American Spanish    
# bn   : Bengali                  lv   : Latvian                   
# pt_BR: Brazilian Portuguese     lt   : Lithuanian                
# bg   : Bulgarian                ms   : Malay                     
# fr_CA: Canadian French          es_MX: Mexican Spanish           
# ca   : Catalan                  no   : Norwegian                 
# hr   : Croatian                 pl   : Polish                    
# cs   : Czech                    pt   : Portuguese                
# da   : Danish                   ro   : Romanian                  
# nl   : Dutch                    ru   : Russian                   
# ar_EG: Egyptian                 zh_CN: Simplified Chinese        
# en_GB: English (Great Britain)  sk   : Slovak                    
# et   : Estonian                 sl   : Slovenian                 
# fi   : Finnish                  es_ES: Spanish                   
# de   : German                   sv   : Swedish                   
# el   : Greek                    th   : Thai                      
# iw   : Hebrew                   zh_TW: Traditional Chinese       
# hu   : Hungarian                tr   : Turkish                   
# is   : Icelandic                uk   : Ukrainian                 
# in   : Indonesian               vi   : Vietnamese                
# it   : Italian                                                   
#
# all_langs   : All languages
#
# Specify value as the following to select any of the languages.
# Example : SELECTED_LANGUAGES=en,fr,ja
#
# Specify value as the following to select all the languages.
# Example : SELECTED_LANGUAGES=all_langs 
# Steve  : 等于图形界面选语言那里
#------------------------------------------------------------------------------
SELECTED_LANGUAGES=en,zh_CN

#------------------------------------------------------------------------------
# Specify the complete path of the Oracle Home.
# Steve  : 必填了，按之前规划填吧
#------------------------------------------------------------------------------
ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1

#------------------------------------------------------------------------------
# Specify the complete path of the Oracle Base. 
# Steve  : 必填了，按之前规划填吧
#------------------------------------------------------------------------------
ORACLE_BASE=/u01/app/oracle

#------------------------------------------------------------------------------
# Specify the installation edition of the component.                        
#                                                             
# The value should contain only one of these choices.        
# EE     : Enterprise Edition                                
# SE     : Standard Edition                                  
# SEONE  : Standard Edition One
# PE     : Personal Edition (WINDOWS ONLY)
# Steve  : 选版本，EE
#------------------------------------------------------------------------------
oracle.install.db.InstallEdition=EE

#------------------------------------------------------------------------------
# This variable is used to enable or disable custom install and is considered
# only if InstallEdition is EE.
#
# true  : Components mentioned as part of 'optionalComponents' property
#         are considered for install.
# false : Value for 'optionalComponents' is not considered.
#------------------------------------------------------------------------------
oracle.install.db.EEOptionsSelection=false

#------------------------------------------------------------------------------
# This variable is considered only if 'EEOptionsSelection' is set to true. 
#
# Description: List of Enterprise Edition Options you would like to enable.
#
#              The following choices are available. You may specify any
#              combination of these choices.  The components you choose should
#              be specified in the form "internal-component-name:version"
#              Below is a list of components you may specify to enable.
#        
#              oracle.oraolap:11.2.0.3.0 - Oracle OLAP
#              oracle.rdbms.dm:11.2.0.3.0 - Oracle Data Mining
#              oracle.rdbms.dv:11.2.0.3.0 - Oracle Database Vault
#              oracle.rdbms.lbac:11.2.0.3.0 - Oracle Label Security
#              oracle.rdbms.partitioning:11.2.0.3.0 - Oracle Partitioning
#              oracle.rdbms.rat:11.2.0.3.0 - Oracle Real Application Testing
#------------------------------------------------------------------------------
oracle.install.db.optionalComponents=oracle.rdbms.partitioning:11.2.0.3.0,oracle.oraolap:11.2.0.3.0,oracle.rdbms.dm:11.2.0.3.0,oracle.rdbms.dv:11.2.0.3.0,oracle.rdbms.lbac:11.2.0.3.0,oracle.rdbms.rat:11.2.0.3.0

###############################################################################
#                                                                             #
# PRIVILEGED OPERATING SYSTEM GROUPS                                          #
# ------------------------------------------                                  #
# Provide values for the OS groups to which OSDBA and OSOPER privileges       #
# needs to be granted. If the install is being performed as a member of the   #
# group "dba", then that will be used unless specified otherwise below.       #
#                                                                             #
# The value to be specified for OSDBA and OSOPER group is only for UNIX based #
# Operating System.                                                           #
#                                                                             #
###############################################################################

#------------------------------------------------------------------------------
# The DBA_GROUP is the OS group which is to be granted OSDBA privileges.
# Steve  : 填之前创建得dba
#------------------------------------------------------------------------------
oracle.install.db.DBA_GROUP=dba

#------------------------------------------------------------------------------
# The OPER_GROUP is the OS group which is to be granted OSOPER privileges.
# The value to be specified for OSOPER group is optional.
# Steve  : 填之前创建得oinstall
#------------------------------------------------------------------------------
oracle.install.db.OPER_GROUP=oinstall

#------------------------------------------------------------------------------
# Specify the cluster node names selected during the installation.
# Example : oracle.install.db.CLUSTER_NODES=node1,node2
#------------------------------------------------------------------------------
oracle.install.db.CLUSTER_NODES=

#------------------------------------------------------------------------------
# This variable is used to enable or disable RAC One Node install.
#
# true  : Value of RAC One Node service name is used.
# false : Value of RAC One Node service name is not used.
#
# If left blank, it will be assumed to be false
#------------------------------------------------------------------------------
oracle.install.db.isRACOneInstall=

#------------------------------------------------------------------------------
# Specify the name for RAC One Node Service. 
#------------------------------------------------------------------------------
oracle.install.db.racOneServiceName=

#------------------------------------------------------------------------------
# Specify the type of database to create.
# It can be one of the following:
# - GENERAL_PURPOSE/TRANSACTION_PROCESSING             
# - DATA_WAREHOUSE 
# Steve  : 一般用途
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.type=

#------------------------------------------------------------------------------
# Specify the Starter Database Global Database Name. 
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.globalDBName=

#------------------------------------------------------------------------------
# Specify the Starter Database SID.
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.SID=

#------------------------------------------------------------------------------
# Specify the Starter Database character set.
#                                              
# It can be one of the following:
# AL32UTF8, WE8ISO8859P15, WE8MSWIN1252, EE8ISO8859P2,
# EE8MSWIN1250, NE8ISO8859P10, NEE8ISO8859P4, BLT8MSWIN1257,
# BLT8ISO8859P13, CL8ISO8859P5, CL8MSWIN1251, AR8ISO8859P6,
# AR8MSWIN1256, EL8ISO8859P7, EL8MSWIN1253, IW8ISO8859P8,
# IW8MSWIN1255, JA16EUC, JA16EUCTILDE, JA16SJIS, JA16SJISTILDE,
# KO16MSWIN949, ZHS16GBK, TH8TISASCII, ZHT32EUC, ZHT16MSWIN950,
# ZHT16HKSCS, WE8ISO8859P9, TR8MSWIN1254, VN8MSWIN1258
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.characterSet=AL32UTF8

#------------------------------------------------------------------------------
# This variable should be set to true if Automatic Memory Management 
# in Database is desired.
# If Automatic Memory Management is not desired, and memory allocation
# is to be done manually, then set it to false.
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.memoryOption=true

#------------------------------------------------------------------------------
# Specify the total memory allocation for the database. Value(in MB) should be
# at least 256 MB, and should not exceed the total physical memory available 
# on the system.
# Example: oracle.install.db.config.starterdb.memoryLimit=512
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.memoryLimit=

#------------------------------------------------------------------------------
# This variable controls whether to load Example Schemas onto
# the starter database or not.
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.installExampleSchemas=false

#------------------------------------------------------------------------------
# This variable includes enabling audit settings, configuring password profiles
# and revoking some grants to public. These settings are provided by default. 
# These settings may also be disabled.    
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.enableSecuritySettings=true

###############################################################################
#                                                                             #
# Passwords can be supplied for the following four schemas in the             #
# starter database:                                                           #
#   SYS                                                                       #
#   SYSTEM                                                                    #
#   SYSMAN (used by Enterprise Manager)                                       #
#   DBSNMP (used by Enterprise Manager)                                       #
#                                                                             #
# Same password can be used for all accounts (not recommended)                #
# or different passwords for each account can be provided (recommended)       #
#                                                                             #
###############################################################################

#------------------------------------------------------------------------------
# This variable holds the password that is to be used for all schemas in the
# starter database.
# Steve  : 设置全部的密码，不过我们只安装数据库软件，其实上下很多参数没用到
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.password.ALL=<password>

#-------------------------------------------------------------------------------
# Specify the SYS password for the starter database.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.password.SYS=

#-------------------------------------------------------------------------------
# Specify the SYSTEM password for the starter database.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.password.SYSTEM=

#-------------------------------------------------------------------------------
# Specify the SYSMAN password for the starter database.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.password.SYSMAN=

#-------------------------------------------------------------------------------
# Specify the DBSNMP password for the starter database.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.password.DBSNMP=

#-------------------------------------------------------------------------------
# Specify the management option to be selected for the starter database. 
# It can be one of the following:
# 1. GRID_CONTROL
# 2. DB_CONTROL
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.control=DB_CONTROL

#-------------------------------------------------------------------------------
# Specify the Management Service to use if Grid Control is selected to manage 
# the database.      
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.gridcontrol.gridControlServiceURL=

###############################################################################
#                                                                             #
# SPECIFY BACKUP AND RECOVERY OPTIONS                                         #
# ------------------------------------                                        #
# Out-of-box backup and recovery options for the database can be mentioned    #
# using the entries below.                                                    #
#                                                                             #
###############################################################################

#------------------------------------------------------------------------------
# This variable is to be set to false if automated backup is not required. Else 
# this can be set to true.
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.automatedBackup.enable=false

#------------------------------------------------------------------------------
# Regardless of the type of storage that is chosen for backup and recovery, if 
# automated backups are enabled, a job will be scheduled to run daily to backup 
# the database. This job will run as the operating system user that is 
# specified in this variable.
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.automatedBackup.osuid=

#-------------------------------------------------------------------------------
# Regardless of the type of storage that is chosen for backup and recovery, if 
# automated backups are enabled, a job will be scheduled to run daily to backup 
# the database. This job will run as the operating system user specified by the 
# above entry. The following entry stores the password for the above operating 
# system user.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.automatedBackup.ospwd=

#-------------------------------------------------------------------------------
# Specify the type of storage to use for the database.
# It can be one of the following:
# - FILE_SYSTEM_STORAGE
# - ASM_STORAGE
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.storageType=

#-------------------------------------------------------------------------------
# Specify the database file location which is a directory for datafiles, control
# files, redo logs.         
#
# Applicable only when oracle.install.db.config.starterdb.storage=FILE_SYSTEM_STORAGE 
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.fileSystemStorage.dataLocation=

#-------------------------------------------------------------------------------
# Specify the backup and recovery location.
#
# Applicable only when oracle.install.db.config.starterdb.storage=FILE_SYSTEM_STORAGE 
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.fileSystemStorage.recoveryLocation=

#-------------------------------------------------------------------------------
# Specify the existing ASM disk groups to be used for storage.
#
# Applicable only when oracle.install.db.config.starterdb.storage=ASM_STORAGE
#-------------------------------------------------------------------------------
oracle.install.db.config.asm.diskGroup=

#-------------------------------------------------------------------------------
# Specify the password for ASMSNMP user of the ASM instance.                  
#
# Applicable only when oracle.install.db.config.starterdb.storage=ASM_STORAGE 
#-------------------------------------------------------------------------------
oracle.install.db.config.asm.ASMSNMPPassword=

#------------------------------------------------------------------------------
# Specify the My Oracle Support Account Username.
#
#  Example   : MYORACLESUPPORT_USERNAME=abc@oracle.com
# Steve  : SUPPORT帐号，不用输入
#------------------------------------------------------------------------------
MYORACLESUPPORT_USERNAME=

#------------------------------------------------------------------------------
# Specify the My Oracle Support Account Username password.
#
# Example    : MYORACLESUPPORT_PASSWORD=password
#------------------------------------------------------------------------------
MYORACLESUPPORT_PASSWORD=

#------------------------------------------------------------------------------
# Specify whether to enable the user to set the password for
# My Oracle Support credentials. The value can be either true or false.
# If left blank it will be assumed to be false.
#
# Example    : SECURITY_UPDATES_VIA_MYORACLESUPPORT=true
#------------------------------------------------------------------------------
SECURITY_UPDATES_VIA_MYORACLESUPPORT=

#------------------------------------------------------------------------------
# Specify whether user doesn't want to configure Security Updates.
# The value for this variable should be true if you don't want to configure
# Security Updates, false otherwise. 
#
# The value can be either true or false. If left blank it will be assumed
# to be false.
#
# Example    : DECLINE_SECURITY_UPDATES=false
# Steve  : 这里有点Tricky，是选true才不会用配置更新补丁，但默认是false，默认执行会报错。
#------------------------------------------------------------------------------
DECLINE_SECURITY_UPDATES=TRUE

#------------------------------------------------------------------------------
# Specify the Proxy server name. Length should be greater than zero.
#
# Example    : PROXY_HOST=proxy.domain.com 
#------------------------------------------------------------------------------
PROXY_HOST=

#------------------------------------------------------------------------------
# Specify the proxy port number. Should be Numeric and atleast 2 chars.
#
# Example    : PROXY_PORT=25 
#------------------------------------------------------------------------------
PROXY_PORT=

#------------------------------------------------------------------------------
# Specify the proxy user name. Leave PROXY_USER and PROXY_PWD 
# blank if your proxy server requires no authentication.
#
# Example    : PROXY_USER=username 
#------------------------------------------------------------------------------
PROXY_USER=

#------------------------------------------------------------------------------
# Specify the proxy password. Leave PROXY_USER and PROXY_PWD  
# blank if your proxy server requires no authentication.
#
# Example    : PROXY_PWD=password 
#------------------------------------------------------------------------------
PROXY_PWD=

#------------------------------------------------------------------------------
# Specify the proxy realm. This value is used if auto-updates option is selected.
#
# Example    : PROXY_REALM=metalink 
#------------------------------------------------------------------------------
PROXY_REALM=

#------------------------------------------------------------------------------
# Specify the Oracle Support Hub URL. 
# 
# Example    : COLLECTOR_SUPPORTHUB_URL=https://orasupporthub.company.com:8080/
#------------------------------------------------------------------------------
COLLECTOR_SUPPORTHUB_URL=

#------------------------------------------------------------------------------
# Specify the auto-updates option. It can be one of the following:
# a.MYORACLESUPPORT_DOWNLOAD
# b.OFFLINE_UPDATES
# c.SKIP_UPDATES
#------------------------------------------------------------------------------
oracle.installer.autoupdates.option=
#------------------------------------------------------------------------------
# In case MYORACLESUPPORT_DOWNLOAD option is chosen, specify the location where
# the updates are to be downloaded.
# In case OFFLINE_UPDATES option is chosen, specify the location where the updates 
# are present.
oracle.installer.autoupdates.downloadUpdatesLoc=
#------------------------------------------------------------------------------
# Specify the My Oracle Support Account Username which has the patches download privileges  
# to be used for software updates.
#  Example   : AUTOUPDATES_MYORACLESUPPORT_USERNAME=abc@oracle.com
#------------------------------------------------------------------------------
AUTOUPDATES_MYORACLESUPPORT_USERNAME=

#------------------------------------------------------------------------------
# Specify the My Oracle Support Account Username password which has the patches download privileges  
# to be used for software updates.
#
# Example    : AUTOUPDATES_MYORACLESUPPORT_PASSWORD=password
#------------------------------------------------------------------------------
AUTOUPDATES_MYORACLESUPPORT_PASSWORD=
```
</details>

设置完以后，运行
```
./runInstaller -silent -responseFile /u01/app/oracle/db_install.rsp
```
另起一个窗口可以***tail***一下生成的日志文件，进行检查和排错

运行完以后会提示需要使用Root执行两个命令，这个和图形化安装是一样的。
```shell
/home/oracle/oraInventory/orainstRoot.sh
/u01/app/oracle/product/11.2.0/dbhome_1/root.sh
```

### 配置数据库实例
来看看dbca.rsp，同样中文都是我后添加的，使用的时候注意一下。有些内容是Rac才使用，单实例可以不设置，注意一下默认值，使用的时候记得去掉#号。

简化版内容如下

```shell
##############################################################################
##  Simple Response File for Install Single Instance DB  Only               ##
##############################################################################

[GENERAL]
RESPONSEFILE_VERSION = "11.2.0"
OPERATION_TYPE = "createDatabase"
[CREATEDATABASE]
GDBNAME = "WMSTDB"
SID = "WMSTDB"
TEMPLATENAME = "General_Purpose.dbc"
SYSPASSWORD = "welcome1"
SYSTEMPASSWORD = "welcome1"
CHARACTERSET = "AL32UTF8"
NATIONALCHARACTERSET= "AL16UTF16"
LISTENERS = "listener"
MEMORYPERCENTAGE = "75"
AUTOMATICMEMORYMANAGEMENT = "TRUE"
TOTALMEMORY = "24576"
```

<details>
<summary><font color='#ea6f5a'><b><i>单击展开 dbca.rsp详细内容</i></b></font></summary>

```shell
[root@WMS-ZJK-DB response]# cat dbca.rsp
##############################################################################
##                                                                          ##
##                            DBCA response file                            ##
##                            ------------------                            ##
## Copyright   1998, 2011, Oracle Corporation. All Rights Reserved.         ##
##                                                                          ##
## Specify values for the variables listed below to customize Oracle        ##
## Database Configuration installation.                                     ##
##                                                                          ##
## Each variable is associated with a comment. The comment identifies the   ##
## variable type.                                                           ##
##                                                                          ##
## Please specify the values in the following format :                      ##
##          Type       :  Example                                           ##
##          String     :  "<value>"                                         ##
##          Boolean    :  True or False                                     ##
##          Number     :  <numeric value>                                   ##
##          StringList :  {"<value1>","<value2>"}                           ##
##                                                                          ##
## Examples :                                                               ## 
##     1. dbca -progress_only -responseFile <response file>                 ##
##        Display a progress bar depicting progress of database creation    ##
##        process.                                                          ##
##                                                                          ##
##     2. dbca -silent -responseFile <response file>                        ## 
##        Creates database silently. No user interface is displayed.        ##
##                                                                          ##
##     3. dbca -silent -createDatabase -cloneTemplate                       ##
##                       -responseFile <response file>                      ##
##        Creates database silently with clone template. The template in    ##
##        responsefile is a clone template.                                 ##
##                                                                          ##
##     4. dbca -silent -deleteDatabase -responseFile <response file>        ##
##        Deletes database silently.                                        ##
##############################################################################

#-----------------------------------------------------------------------------
# GENERAL section is required for all types of database creations.
#-----------------------------------------------------------------------------
[GENERAL]

#-----------------------------------------------------------------------------
# Name          : RESPONSEFILE_VERSION
# Datatype      : String
# Description   : Version of the database to create
# Valid values  : "11.1.0"
# Default value : None
# Mandatory     : Yes
#-----------------------------------------------------------------------------
RESPONSEFILE_VERSION = "11.2.0"

#-----------------------------------------------------------------------------
# Name          : OPERATION_TYPE
# Datatype      : String
# Description   : Type of operation
# Valid values  : "createDatabase" \ "createTemplateFromDB" \ "createCloneTemplate" \ "deleteDatabase" \ "configureDatabase" \ "addInstance" (RAC-only) \ "deleteInstance" (RAC-only)
# Default value : None
# Mandatory     : Yes
# Steve         : 选择安装类型，仅安装数据库软件，对应的后面只用看createDatabase section。
#-----------------------------------------------------------------------------
OPERATION_TYPE = "createDatabase"

#-----------------------*** End of GENERAL section ***------------------------

#-----------------------------------------------------------------------------
# CREATEDATABASE section is used when OPERATION_TYPE is defined as "createDatabase". 
#-----------------------------------------------------------------------------
[CREATEDATABASE]

#-----------------------------------------------------------------------------
# Name          : GDBNAME
# Datatype      : String
# Description   : Global database name of the database
# Valid values  : <db_name>.<db_domain> - when database domain isn't NULL
#                 <db_name>             - when database domain is NULL
# Default value : None
# Mandatory     : Yes
# Steve         : 设置Global Name
#-----------------------------------------------------------------------------
GDBNAME = "<ORACLE_GLOBAL_NAME>"

#-----------------------------------------------------------------------------
# Name          : RACONENODE
# Datatype      : Boolean
# Description   : Set to true for RAC One Node database
# Valid values  : TRUE\FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#RACONENODE  = "false"

#-----------------------------------------------------------------------------
# Name          : RACONENODESERVICENAME
# Datatype      : String
# Description   : Service is required by application to connect to RAC One 
#                 Node Database
# Valid values  : Service Name
# Default value : None
# Mandatory     : No [required in case RACONENODE flag is set to true]
#-----------------------------------------------------------------------------
#RACONENODESERVICENAME = 

#-----------------------------------------------------------------------------
# Name          : POLICYMANAGED
# Datatype      : Boolean
# Description   : Set to true if Database is policy managed and 
#                 set to false if  Database is admin managed
# Valid values  : TRUE\FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#POLICYMANAGED = "false"

#-----------------------------------------------------------------------------
# Name          : CREATESERVERPOOL
# Datatype      : Boolean
# Description   : Set to true if new server pool need to be created for database 
#                 if this option is specified then the newly created database 
#                 will use this newly created serverpool. 
#                 Multiple serverpoolname can not be specified for database
# Valid values  : TRUE\FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#CREATESERVERPOOL = "false"

#-----------------------------------------------------------------------------
# Name          : FORCE
# Datatype      : Boolean
# Description   : Set to true if new server pool need to be created by force 
#                 if this option is specified then the newly created serverpool
#                 will be assigned server even if no free servers are available.
#                 This may affect already running database.
#                 This flag can be specified for Admin managed as well as policy managed db.
# Valid values  : TRUE\FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#FORCE = "false"

#-----------------------------------------------------------------------------
# Name          : SERVERPOOLNAME
# Datatype      : String
# Description   : Only one serverpool name need to be specified 
#                  if Create Server Pool option is specified. 
#                  Comma-separated list of Serverpool names if db need to use
#                  multiple Server pool
# Valid values  : ServerPool name
# Default value : None
# Mandatory     : No [required in case of RAC service centric database]
#-----------------------------------------------------------------------------
#SERVERPOOLNAME = 

#-----------------------------------------------------------------------------
# Name          : CARDINALITY
# Datatype      : Number
# Description   : Specify Cardinality for create server pool operation
# Valid values  : any positive Integer value
# Default value : Number of qualified nodes on cluster
# Mandatory     : No [Required when a new serverpool need to be created]
#-----------------------------------------------------------------------------
#CARDINALITY = 

#-----------------------------------------------------------------------------
# Name          : SID
# Datatype      : String
# Description   : System identifier (SID) of the database
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : <db_name> specified in GDBNAME
# Mandatory     : No
# Steve         : 设置SID。
#-----------------------------------------------------------------------------
SID = "<SID>"

#-----------------------------------------------------------------------------
# Name          : NODELIST
# Datatype      : String
# Description   : Comma-separated list of cluster nodes
# Valid values  : Cluster node names
# Default value : None
# Mandatory     : No (Yes for RAC database-centric database )
#-----------------------------------------------------------------------------
#NODELIST=

#-----------------------------------------------------------------------------
# Name          : TEMPLATENAME
# Datatype      : String
# Description   : Name of the template
# Valid values  : Template file name
# Default value : None
# Mandatory     : Yes
#-----------------------------------------------------------------------------
TEMPLATENAME = "General_Purpose.dbc"

#-----------------------------------------------------------------------------
# Name          : OBFUSCATEDPASSWORDS
# Datatype      : Boolean
# Description   : Set to true if passwords are encrypted
# Valid values  : TRUE\FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#OBFUSCATEDPASSWORDS = FALSE


#-----------------------------------------------------------------------------
# Name          : SYSPASSWORD
# Datatype      : String
# Description   : Password for SYS user
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : Yes
# Steve         : SYS的密码,建议设置一下。
#-----------------------------------------------------------------------------
SYSPASSWORD = "<password>"

#-----------------------------------------------------------------------------
# Name          : SYSTEMPASSWORD
# Datatype      : String
# Description   : Password for SYSTEM user
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : Yes
# Steve         : SYSTEM的密码，建议设置一下。
#-----------------------------------------------------------------------------
SYSTEMPASSWORD = "<password>"

#-----------------------------------------------------------------------------
# Name          : EMCONFIGURATION
# Datatype      : String
# Description   : Enterprise Manager Configuration Type
# Valid values  : CENTRAL|LOCAL|ALL|NONE
# Default value : NONE
# Mandatory     : No
# Steve         : 开不开EM
#-----------------------------------------------------------------------------
#EMCONFIGURATION = "NONE"

#-----------------------------------------------------------------------------
# Name          : DISABLESECURITYCONFIGURATION
# Datatype      : String
# Description   : Database Security Settings
# Valid values  : ALL|NONE|AUDIT|PASSWORD_PROFILE
# Default value : NONE
# Mandatory     : No
#-----------------------------------------------------------------------------
#DISABLESECURITYCONFIGURATION = "NONE"


#-----------------------------------------------------------------------------
# Name          : SYSMANPASSWORD
# Datatype      : String
# Description   : Password for SYSMAN user
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : Yes, if LOCAL specified for EMCONFIGURATION
#-----------------------------------------------------------------------------
#SYSMANPASSWORD = "password"

#-----------------------------------------------------------------------------
# Name          : DBSNMPPASSWORD
# Datatype      : String
# Description   : Password for DBSNMP user
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : Yes, if EMCONFIGURATION is specified
#-----------------------------------------------------------------------------
#DBSNMPPASSWORD = "password"

#-----------------------------------------------------------------------------
# Name          : CENTRALAGENT
# Datatype      : String
# Description   : Grid Control Central Agent Oracle Home
# Default value : None
# Mandatory     : Yes, if CENTRAL is specified for EMCONFIGURATION
#-----------------------------------------------------------------------------
#CENTRALAGENT = 

#-----------------------------------------------------------------------------
# Name          : HOSTUSERNAME
# Datatype      : String
# Description   : Host user name for EM backup job
# Default value : None
# Mandatory     : Yes, if ALL is specified for EMCONFIGURATION
#-----------------------------------------------------------------------------
#HOSTUSERNAME = 

#-----------------------------------------------------------------------------
# Name          : HOSTUSERPASSWORD
# Datatype      : String
# Description   : Host user password for EM backup job
# Default value : None
# Mandatory     : Yes, if ALL is specified for EMCONFIGURATION
#-----------------------------------------------------------------------------
#HOSTUSERPASSWORD= 

#-----------------------------------------------------------------------------
# Name          : BACKUPSCHEDULE
# Datatype      : String
# Description   : Daily backup schedule in the form of hh:mm
# Default value : 2:00
# Mandatory     : Yes, if ALL is specified for EMCONFIGURATION
#-----------------------------------------------------------------------------
#BACKUPSCHEDULE=

#-----------------------------------------------------------------------------
# Name          : DVOWNERNAME
# Datatype      : String
# Description   : DataVault Owner
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : Yes, if DataVault option is chosen
#-----------------------------------------------------------------------------
#DVOWNERNAME = ""

#-----------------------------------------------------------------------------
# Name          : DVOWNERPASSWORD
# Datatype      : String
# Description   : Password for DataVault Owner
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : Yes, if DataVault option is chosen
#-----------------------------------------------------------------------------
#DVOWNERPASSWORD = ""

#-----------------------------------------------------------------------------
# Name          : DVACCOUNTMANAGERNAME
# Datatype      : String
# Description   : DataVault Account Manager
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : No
#-----------------------------------------------------------------------------
#DVACCOUNTMANAGERNAME = ""

#-----------------------------------------------------------------------------
# Name          : DVACCOUNTMANAGERPASSWORD
# Datatype      : String
# Description   : Password for  DataVault Account Manager
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : No
#-----------------------------------------------------------------------------
#DVACCOUNTMANAGERPASSWORD = ""



#-----------------------------------------------------------------------------
# Name          : DATAFILEJARLOCATION 
# Datatype      : String
# Description   : Location of the data file jar 
# Valid values  : Directory containing compressed datafile jar
# Default value : None
# Mandatory     : No
#-----------------------------------------------------------------------------
#DATAFILEJARLOCATION =

#-----------------------------------------------------------------------------
# Name          : DATAFILEDESTINATION 
# Datatype      : String
# Description   : Location of the data file's
# Valid values  : Directory for all the database files
# Default value : $ORACLE_BASE/oradata
# Mandatory     : No
# Steve         : 数据文件的目录了，就是system01.dbf那些，看自己喜欢设置。
#-----------------------------------------------------------------------------
#DATAFILEDESTINATION =

#-----------------------------------------------------------------------------
# Name          : RECOVERYAREADESTINATION
# Datatype      : String
# Description   : Location of the data file's
# Valid values  : Recovery Area location
# Default value : $ORACLE_BASE/flash_recovery_area
# Mandatory     : No
# Steve         : 快速闪回区的设置，默认是由，可以建好库以后再修改归档路径和闪回区参数去掉。
#-----------------------------------------------------------------------------
#RECOVERYAREADESTINATION=

#-----------------------------------------------------------------------------
# Name          : STORAGETYPE
# Datatype      : String
# Description   : Specifies the storage on which the database is to be created
# Valid values  : FS (CFS for RAC), ASM
# Default value : FS
# Mandatory     : No
#-----------------------------------------------------------------------------
#STORAGETYPE=FS

#-----------------------------------------------------------------------------
# Name          : DISKGROUPNAME
# Datatype      : String
# Description   : Specifies the disk group name for the storage
# Default value : DATA
# Mandatory     : No
# Steve         : Rac用的
#-----------------------------------------------------------------------------
#DISKGROUPNAME=DATA

#-----------------------------------------------------------------------------
# Name          : ASMSNMP_PASSWORD
# Datatype      : String
# Description   : Password for ASM Monitoring
# Default value : None
# Mandatory     : No
#-----------------------------------------------------------------------------
#ASMSNMP_PASSWORD=""

#-----------------------------------------------------------------------------
# Name          : RECOVERYGROUPNAME
# Datatype      : String
# Description   : Specifies the disk group name for the recovery area
# Default value : RECOVERY
# Mandatory     : No
#-----------------------------------------------------------------------------
#RECOVERYGROUPNAME=RECOVERY


#-----------------------------------------------------------------------------
# Name          : CHARACTERSET
# Datatype      : String
# Description   : Character set of the database
# Valid values  : Check Oracle11g National Language Support Guide
# Default value : "US7ASCII"
# Mandatory     : NO
# Steve         : 字符集，记得改自己想要的。
#-----------------------------------------------------------------------------
CHARACTERSET = "AL32UTF8"

#-----------------------------------------------------------------------------
# Name          : NATIONALCHARACTERSET
# Datatype      : String
# Description   : National Character set of the database
# Valid values  : "UTF8" or "AL16UTF16". For details, check Oracle11g National Language Support Guide
# Default value : "AL16UTF16"
# Mandatory     : No
# Steve         : 国家字符集，记得改自己想要的
#-----------------------------------------------------------------------------
NATIONALCHARACTERSET= "AL16UTF16"

#-----------------------------------------------------------------------------
# Name          : REGISTERWITHDIRSERVICE
# Datatype      : Boolean
# Description   : Specifies whether to register with Directory Service.
# Valid values  : TRUE \ FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#REGISTERWITHDIRSERVICE= TRUE

#-----------------------------------------------------------------------------
# Name          : DIRSERVICEUSERNAME
# Datatype      : String
# Description   : Specifies the name of the directory service user
# Mandatory     : YES, if the value of registerWithDirService is TRUE
#-----------------------------------------------------------------------------
#DIRSERVICEUSERNAME= "name"

#-----------------------------------------------------------------------------
# Name          : DIRSERVICEPASSWORD
# Datatype      : String
# Description   : The password of the directory service user.
#                 You can also specify the password at the command prompt instead of here.
# Mandatory     : YES, if the value of registerWithDirService is TRUE
#-----------------------------------------------------------------------------
#DIRSERVICEPASSWORD= "password"

#-----------------------------------------------------------------------------
# Name          : WALLETPASSWORD
# Datatype      : String
# Description   : The password for wallet to created or modified.
#                 You can also specify the password at the command prompt instead of here.
# Mandatory     : YES, if the value of registerWithDirService is TRUE
#-----------------------------------------------------------------------------
#WALLETPASSWORD= "password"

#-----------------------------------------------------------------------------
# Name          : LISTENERS
# Datatype      : String
# Description   : Specifies list of listeners to register the database with.
#                 By default the database is configured for all the listeners specified in the 
#                 $ORACLE_HOME/network/admin/listener.ora 
# Valid values  : The list should be space separated names like "listener1 listener2".
# Mandatory     : NO
# Steve         : 通常设置一个listener就行了，后面建listener的时候也是如此。
#-----------------------------------------------------------------------------
LISTENERS = "listener"

#-----------------------------------------------------------------------------
# Name          : VARIABLESFILE 
# Datatype      : String
# Description   : Location of the file containing variable value pair
# Valid values  : A valid file-system file. The variable value pair format in this file 
#                 is <variable>=<value>. Each pair should be in a new line.
# Default value : None
# Mandatory     : NO
#-----------------------------------------------------------------------------
#VARIABLESFILE =

#-----------------------------------------------------------------------------
# Name          : VARIABLES
# Datatype      : String
# Description   : comma separated list of name=value pairs. Overrides variables defined in variablefile and templates
# Default value : None
# Mandatory     : NO
#-----------------------------------------------------------------------------
#VARIABLES =

#-----------------------------------------------------------------------------
# Name          : INITPARAMS
# Datatype      : String
# Description   : comma separated list of name=value pairs. Overrides initialization parameters defined in templates
# Default value : None
# Mandatory     : NO
#-----------------------------------------------------------------------------
#INITPARAMS =

#-----------------------------------------------------------------------------
# Name          : SAMPLESCHEMA
# Datatype      : Boolean
# Description   : Specifies whether or not to add the Sample Schemas to your database
# Valid values  : TRUE \ FALSE
# Default value : FASLE
# Mandatory     : No
#-----------------------------------------------------------------------------
#SAMPLESCHEMA=TRUE

#-----------------------------------------------------------------------------
# Name          : MEMORYPERCENTAGE
# Datatype      : String
# Description   : percentage of physical memory for Oracle
# Default value : None
# Mandatory     : NO
# Steve         : 内存使用100%，我一般75%
#-----------------------------------------------------------------------------
MEMORYPERCENTAGE = "75"

#-----------------------------------------------------------------------------
# Name          : DATABASETYPE
# Datatype      : String
# Description   : used for memory distribution when MEMORYPERCENTAGE specified
# Valid values  : MULTIPURPOSE|DATA_WAREHOUSING|OLTP
# Default value : MULTIPURPOSE
# Mandatory     : NO
#-----------------------------------------------------------------------------
#DATABASETYPE = "MULTIPURPOSE"

#-----------------------------------------------------------------------------
# Name          : AUTOMATICMEMORYMANAGEMENT
# Datatype      : Boolean
# Description   : flag to indicate Automatic Memory Management is used
# Valid values  : TRUE/FALSE
# Default value : TRUE
# Mandatory     : NO
# Steve         : 默认启用AMM
#-----------------------------------------------------------------------------
#AUTOMATICMEMORYMANAGEMENT = "TRUE"

#-----------------------------------------------------------------------------
# Name          : TOTALMEMORY
# Datatype      : String
# Description   : total memory in MB to allocate to Oracle
# Valid values  : 
# Default value : 
# Mandatory     : NO
# Steve         : 内存使用大小，可以算一下
#-----------------------------------------------------------------------------
TOTALMEMORY = "<75% of RAM>"

# Steve         : 再往下是dbca创建其他使用的参数了，可以暂时不看了。
#-----------------------*** End of CREATEDATABASE section ***------------------------

#-----------------------------------------------------------------------------
# createTemplateFromDB section is used when OPERATION_TYPE is defined as "createTemplateFromDB". 
#-----------------------------------------------------------------------------
[createTemplateFromDB]
#-----------------------------------------------------------------------------
# Name          : SOURCEDB 
# Datatype      : String
# Description   : The source database from which to create the template
# Valid values  : The format is <host>:<port>:<sid>
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
SOURCEDB = "myhost:1521:orcl"

#-----------------------------------------------------------------------------
# Name          : SYSDBAUSERNAME 
# Datatype      : String
# Description   : A user with DBA role.
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
SYSDBAUSERNAME = "system"

#-----------------------------------------------------------------------------
# Name          : SYSDBAPASSWORD 
# Datatype      : String
# Description   : The password of the DBA user.
#                 You can also specify the password at the command prompt instead of here.
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
#SYSDBAPASSWORD = "password"

#-----------------------------------------------------------------------------
# Name          : TEMPLATENAME
# Datatype      : String
# Description   : Name for the new template.
# Default value : None
# Mandatory     : Yes
#-----------------------------------------------------------------------------
TEMPLATENAME = "My Copy TEMPLATE"

#-----------------------*** End of createTemplateFromDB section ***------------------------

#-----------------------------------------------------------------------------
# createCloneTemplate section is used when OPERATION_TYPE is defined as "createCloneTemplate". 
#-----------------------------------------------------------------------------
[createCloneTemplate]
#-----------------------------------------------------------------------------
# Name          : SOURCEDB
# Datatype      : String
# Description   : The source database is the SID from which to create the template. 
#                 This database must be local and on the same ORACLE_HOME.
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
SOURCEDB = "orcl"

#-----------------------------------------------------------------------------
# Name          : SYSDBAUSERNAME
# Datatype      : String
# Description   : A user with DBA role.
# Default value : none
# Mandatory     : YES, if no OS authentication
#-----------------------------------------------------------------------------
#SYSDBAUSERNAME = "sys"

#-----------------------------------------------------------------------------
# Name          : SYSDBAPASSWORD
# Datatype      : String
# Description   : The password of the DBA user.
#                 You can also specify the password at the command prompt instead of here.
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
#SYSDBAPASSWORD = "password"

#-----------------------------------------------------------------------------
# Name          : TEMPLATENAME
# Datatype      : String
# Description   : Name for the new template.
# Default value : None
# Mandatory     : Yes
#-----------------------------------------------------------------------------
TEMPLATENAME = "My Clone TEMPLATE"

#-----------------------------------------------------------------------------
# Name          : DATAFILEJARLOCATION
# Datatype      : String
# Description   : Location of the data file jar 
# Valid values  : Directory where the new compressed datafile jar will be placed
# Default value : $ORACLE_HOME/assistants/dbca/templates
# Mandatory     : NO
#-----------------------------------------------------------------------------
#DATAFILEJARLOCATION = 

#-----------------------*** End of createCloneTemplate section ***------------------------

#-----------------------------------------------------------------------------
# DELETEDATABASE section is used when DELETE_TYPE is defined as "deleteDatabase". 
#-----------------------------------------------------------------------------
[DELETEDATABASE]
#-----------------------------------------------------------------------------
# Name          : SOURCEDB
# Datatype      : String
# Description   : The source database is the SID 
#                 This database must be local and on the same ORACLE_HOME.
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
SOURCEDB = "orcl"

#-----------------------------------------------------------------------------
# Name          : SYSDBAUSERNAME
# Datatype      : String
# Description   : A user with DBA role.
# Default value : none
# Mandatory     : YES, if no OS authentication
#-----------------------------------------------------------------------------
#SYSDBAUSERNAME = "sys"

#-----------------------------------------------------------------------------
# Name          : SYSDBAPASSWORD
# Datatype      : String
# Description   : The password of the DBA user.
#                 You can also specify the password at the command prompt instead of here.
# Default value : none
# Mandatory     : YES, if no OS authentication
#-----------------------------------------------------------------------------
#SYSDBAPASSWORD = "password"
#-----------------------*** End of deleteDatabase section ***------------------------

#-----------------------------------------------------------------------------
# GENERATESCRIPTS section 
#-----------------------------------------------------------------------------
[generateScripts]
#-----------------------------------------------------------------------------
# Name          : TEMPLATENAME
# Datatype      : String
# Description   : Name of the template
# Valid values  : Template name as seen in DBCA
# Default value : None
# Mandatory     : Yes
#-----------------------------------------------------------------------------
TEMPLATENAME = "New Database"

#-----------------------------------------------------------------------------
# Name          : GDBNAME
# Datatype      : String
# Description   : Global database name of the database
# Valid values  : <db_name>.<db_domain> - when database domain isn't NULL
#                 <db_name>             - when database domain is NULL
# Default value : None
# Mandatory     : Yes
#-----------------------------------------------------------------------------
GDBNAME = "orcl11.us.oracle.com"

#-----------------------------------------------------------------------------
# Name          : SCRIPTDESTINATION 
# Datatype      : String
# Description   : Location of the scripts
# Valid values  : Directory for all the scripts
# Default value : None
# Mandatory     : No
#-----------------------------------------------------------------------------
#SCRIPTDESTINATION =

#-----------------------*** End of deleteDatabase section ***------------------------

#-----------------------------------------------------------------------------
# CONFIGUREDATABASE section is used when OPERATION_TYPE is defined as "configureDatabase". 
#-----------------------------------------------------------------------------
[CONFIGUREDATABASE]

#-----------------------------------------------------------------------------
# Name          : SOURCEDB
# Datatype      : String
# Description   : The source database is the SID 
#                 This database must be local and on the same ORACLE_HOME.
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
#SOURCEDB = "orcl"

#-----------------------------------------------------------------------------
# Name          : SYSDBAUSERNAME
# Datatype      : String
# Description   : A user with DBA role.
# Default value : none
# Mandatory     : YES, if no OS authentication
#-----------------------------------------------------------------------------
#SYSDBAUSERNAME = "sys"


#-----------------------------------------------------------------------------
# Name          : SYSDBAPASSWORD
# Datatype      : String
# Description   : The password of the DBA user.
#                 You can also specify the password at the command prompt instead of here.
# Default value : none
# Mandatory     : YES, if no OS authentication
#-----------------------------------------------------------------------------
#SYSDBAPASSWORD =

#-----------------------------------------------------------------------------
# Name          : REGISTERWITHDIRSERVICE
# Datatype      : Boolean
# Description   : Specifies whether to register with Directory Service.
# Valid values  : TRUE \ FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#REGISTERWITHDIRSERVICE= TRUE

#-----------------------------------------------------------------------------
# Name          : UNREGISTERWITHDIRSERVICE
# Datatype      : Boolean
# Description   : Specifies whether to unregister with Directory Service.
# Valid values  : TRUE \ FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#UNREGISTERWITHDIRSERVICE= TRUE

#-----------------------------------------------------------------------------
# Name          : REGENERATEDBPASSWORD
# Datatype      : Boolean
# Description   : Specifies whether regenerate database password in OID/Wallet
# Valid values  : TRUE \ FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#REGENERATEDBPASSWORD= TRUE

#-----------------------------------------------------------------------------
# Name          : DIRSERVICEUSERNAME
# Datatype      : String
# Description   : Specifies the name of the directory service user
# Mandatory     : YES, if the any of the reg/unreg/regenPasswd options specified
#-----------------------------------------------------------------------------
#DIRSERVICEUSERNAME= "name"

#-----------------------------------------------------------------------------
# Name          : DIRSERVICEPASSWORD
# Datatype      : String
# Description   : The password of the directory service user.
#                 You can also specify the password at the command prompt instead of here.
# Mandatory     : YES, if the any of the reg/unreg/regenPasswd options specified
#-----------------------------------------------------------------------------
#DIRSERVICEPASSWORD= "password"

#-----------------------------------------------------------------------------
# Name          : WALLETPASSWORD
# Datatype      : String
# Description   : The password for wallet to created or modified.
#                 You can also specify the password at the command prompt instead of here.
# Mandatory     : YES, if the any of the reg/unreg/regenPasswd options specified
#-----------------------------------------------------------------------------
#WALLETPASSWORD= "password"

#-----------------------------------------------------------------------------
# Name          : DISABLESECURITYCONFIGURATION
# Datatype      : String
# Description   : Database Security Settings
# Valid values  : ALL|NONE|AUDIT|PASSWORD_PROFILE
# Default value : NONE
# Mandatory     : No
#-----------------------------------------------------------------------------
#DISABLESECURITYCONFIGURATION = "NONE"



#-----------------------------------------------------------------------------
# Name          : ENABLESECURITYCONFIGURATION
# Datatype      : String
# Description   : Database Security Settings
# Valid values  : true|false
# Default value : true
# Mandatory     : No
#-----------------------------------------------------------------------------
#ENABLESECURITYCONFIGURATION = "true"


#-----------------------------------------------------------------------------
# Name          : EMCONFIGURATION
# Datatype      : String
# Description   : Enterprise Manager Configuration Type
# Valid values  : CENTRAL|LOCAL|ALL|NONE
# Default value : NONE
# Mandatory     : No
#-----------------------------------------------------------------------------
#EMCONFIGURATION = "NONE"

#-----------------------------------------------------------------------------
# Name          : SYSMANPASSWORD
# Datatype      : String
# Description   : Password for SYSMAN user
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : Yes, if LOCAL specified for EMCONFIGURATION
#-----------------------------------------------------------------------------
#SYSMANPASSWORD = "password"

#-----------------------------------------------------------------------------
# Name          : DBSNMPPASSWORD
# Datatype      : String
# Description   : Password for DBSNMP user
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : Yes, if EMCONFIGURATION is specified
#-----------------------------------------------------------------------------
#DBSNMPPASSWORD = "password"

#-----------------------------------------------------------------------------
# Name          : CENTRALAGENT
# Datatype      : String
# Description   : Grid Control Central Agent Oracle Home
# Default value : None
# Mandatory     : Yes, if CENTRAL is specified for EMCONFIGURATION
#-----------------------------------------------------------------------------
#CENTRALAGENT = 

#-----------------------------------------------------------------------------
# Name          : HOSTUSERNAME
# Datatype      : String
# Description   : Host user name for EM backup job
# Default value : None
# Mandatory     : Yes, if ALL is specified for EMCONFIGURATION
#-----------------------------------------------------------------------------
#HOSTUSERNAME = 

#-----------------------------------------------------------------------------
# Name          : HOSTUSERPASSWORD
# Datatype      : String
# Description   : Host user password for EM backup job
# Default value : None
# Mandatory     : Yes, if ALL is specified for EMCONFIGURATION
#-----------------------------------------------------------------------------
#HOSTUSERPASSWORD= 

#-----------------------------------------------------------------------------
# Name          : BACKUPSCHEDULE
# Datatype      : String
# Description   : Daily backup schedule in the form of hh:mm
# Default value : 2:00
# Mandatory     : Yes, if ALL is specified for EMCONFIGURATION
#-----------------------------------------------------------------------------
#BACKUPSCHEDULE=

#-----------------------*** End of CONFIGUREDATABASE section ***------------------------


#-----------------------------------------------------------------------------
# ADDINSTANCE section is used when OPERATION_TYPE is defined as "addInstance". 
#-----------------------------------------------------------------------------
[ADDINSTANCE]

#-----------------------------------------------------------------------------
# Name          : DB_UNIQUE_NAME
# Datatype      : String
# Description   : DB Unique Name of the RAC database
# Valid values  : <db_unique_name>
# Default value : None
# Mandatory     : Yes
#-----------------------------------------------------------------------------
DB_UNIQUE_NAME = "orcl11g.us.oracle.com"

#-----------------------------------------------------------------------------
# Name          : INSTANCENAME
# Datatype      : String
# Description   : RAC instance name to be added
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : <sid_prefix>+<highest_current_thread+1>
# Mandatory     : No
#-----------------------------------------------------------------------------
#INSTANCENAME = "orcl1"

#-----------------------------------------------------------------------------
# Name          : NODELIST
# Datatype      : String
# Description   : Node on which to add new instance 
#                 (in 10gR2, instance addition is supported on 1 node at a time)
# Valid values  : Cluster node name
# Default value : None
# Mandatory     : Yes
#-----------------------------------------------------------------------------
NODELIST=

#-----------------------------------------------------------------------------
# Name          : OBFUSCATEDPASSWORDS
# Datatype      : Boolean
# Description   : Set to true if passwords are encrypted
# Valid values  : TRUE\FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#OBFUSCATEDPASSWORDS = FALSE

#-----------------------------------------------------------------------------
# Name          : SYSDBAUSERNAME 
# Datatype      : String
# Description   : A user with DBA role.
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
SYSDBAUSERNAME = "sys"

#-----------------------------------------------------------------------------
# Name          : SYSDBAPASSWORD 
# Datatype      : String
# Description   : The password of the DBA user.
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
#SYSDBAPASSWORD = "password"

#-----------------------*** End of ADDINSTANCE section ***------------------------


#-----------------------------------------------------------------------------
# DELETEINSTANCE section is used when OPERATION_TYPE is defined as "deleteInstance". 
#-----------------------------------------------------------------------------
[DELETEINSTANCE]

#-----------------------------------------------------------------------------
# Name          : DB_UNIQUE_NAME
# Datatype      : String
# Description   : DB Unique Name of the RAC database
# Valid values  : <db_unique_name>
# Default value : None
# Mandatory     : Yes
#-----------------------------------------------------------------------------
DB_UNIQUE_NAME = "orcl11g.us.oracle.com"

#-----------------------------------------------------------------------------
# Name          : INSTANCENAME
# Datatype      : String
# Description   : RAC instance name to be deleted
# Valid values  : Check Oracle11g Administrator's Guide
# Default value : None
# Mandatory     : Yes
#-----------------------------------------------------------------------------
INSTANCENAME = "orcl11g"

#-----------------------------------------------------------------------------
# Name          : NODELIST
# Datatype      : String
# Description   : Node on which instance to be deleted (SID) is located
# Valid values  : Cluster node name
# Default value : None
# Mandatory     : No
#-----------------------------------------------------------------------------
#NODELIST=

#-----------------------------------------------------------------------------
# Name          : OBFUSCATEDPASSWORDS
# Datatype      : Boolean
# Description   : Set to true if passwords are encrypted
# Valid values  : TRUE\FALSE
# Default value : FALSE
# Mandatory     : No
#-----------------------------------------------------------------------------
#OBFUSCATEDPASSWORDS = FALSE

#-----------------------------------------------------------------------------
# Name          : SYSDBAUSERNAME 
# Datatype      : String
# Description   : A user with DBA role.
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
SYSDBAUSERNAME = "sys"

#-----------------------------------------------------------------------------
# Name          : SYSDBAPASSWORD 
# Datatype      : String
# Description   : The password of the DBA user.
# Default value : none
# Mandatory     : YES
#-----------------------------------------------------------------------------
#SYSDBAPASSWORD = "password"


#-----------------------*** End of DELETEINSTANCE section ***------------------------
```
</details>

设置完以后，运行
```shell
dbca -silent -responseFile /u01/app/oracle/dbca.rsp
```
另起一个窗口可以***tail***一下生成的日志文件，进行检查和排错。

到这一步数据库就安装完毕了，可以***sqlplus***登录看看了

### 配置监听
来看看netca.rsp，图形界面创建listener基本也是一路下一步，所以netca.rsp修改的内容确实不多，加上创建以后，可以自己修改listener.ora文件来修改配置，所以影响不大。

简化内容如下：
```shell
################################################
## Simple Response File for NETCA             ##
################################################

[GENERAL]
RESPONSEFILE_VERSION="11.2"
CREATE_TYPE="CUSTOM"

[oracle.net.ca]
INSTALLED_COMPONENTS={"server","net8","javavm"}
INSTALL_TYPE=""typical""
LISTENER_NUMBER=1
LISTENER_NAMES={"LISTENER"}
LISTENER_PROTOCOLS={"TCP;1521"}
LISTENER_START=""LISTENER""
NAMING_METHODS={"TNSNAMES","ONAMES","HOSTNAME"}
NSN_NUMBER=1
NSN_NAMES={"EXTPROC_CONNECTION_DATA"}
NSN_SERVICE={"PLSExtProc"}
NSN_PROTOCOLS={"TCP;HOSTNAME;1521"}
```

<details>
<summary><font color='#ea6f5a'><b><i>单击展开 netca.rsp 详细内容</i></b></font></summary>

```shell
[oracle@WMS-ZJK-DB rsp]$ cat netca.rsp
###################################################################### 
## Copyright(c) 1998, 2011 Oracle Corporation. All rights reserved. ## 
##                                                                  ## 
## Specify values for the variables listed below to customize your  ## 
## installation.                                                    ## 
##                                                                  ## 
## Each variable is associated with a comment. The comment          ## 
## identifies the variable type.                                    ## 
##                                                                  ## 
## Please specify the values in the following format:               ## 
##                                                                  ## 
##         Type         Example                                     ## 
##         String       "Sample Value"                              ## 
##         Boolean      True or False                               ## 
##         Number       1000                                        ## 
##         StringList   {"String value 1","String Value 2"}         ## 
##                                                                  ## 
######################################################################
##                                                                  ## 
## This sample response file causes the Oracle Net Configuration    ##
## Assistant (NetCA) to complete an Oracle Net configuration during ##
## a custom install of the Oracle11g server which is similar to     ##
## what would be created by the NetCA during typical Oracle11g      ##
## install. It also documents all of the NetCA response file        ##
## variables so you can create your own response file to configure  ##
## Oracle Net during an install the way you wish.                   ##
##                                                                  ## 
###################################################################### 

[GENERAL]
RESPONSEFILE_VERSION="11.2"
CREATE_TYPE="CUSTOM"

#-------------------------------------------------------------------------------
# Name       : SHOW_GUI
# Datatype   : Boolean
# Description: This variable controls appearance/suppression of the NetCA GUI,
# Pre-req    : N/A
# Default    : TRUE
# Note:
# This must be set to false in order to run NetCA in silent mode. 
# This is a substitute of "/silent" flag in the NetCA command line.
# The command line flag has precedence over the one in this response file.
# This feature is present since 10.1.0.3.
#-------------------------------------------------------------------------------
#SHOW_GUI=false

#-------------------------------------------------------------------------------
# Name       : LOG_FILE
# Datatype   : String
# Description: If present, NetCA will log output to this file in addition to the
#              standard out.
# Pre-req    : N/A
# Default    : NONE
# Note:
#       This is a substitute of "/log" in the NetCA command line.
# The command line argument has precedence over the one in this response file.
# This feature is present since 10.1.0.3.
#-------------------------------------------------------------------------------
#LOG_FILE=""/oracle11gHome/network/tools/log/netca.log""

[oracle.net.ca]
#INSTALLED_COMPONENTS;StringList;list of installed components
# The possible values for installed components are:
# "net8","server","client","aso", "cman", "javavm" 
INSTALLED_COMPONENTS={"server","net8","javavm"}

#INSTALL_TYPE;String;type of install
# The possible values for install type are:
# "typical","minimal" or "custom"
INSTALL_TYPE=""typical""

#LISTENER_NUMBER;Number;Number of Listeners
# A typical install sets one listener 
LISTENER_NUMBER=1

#LISTENER_NAMES;StringList;list of listener names
# The values for listener are:
# "LISTENER","LISTENER1","LISTENER2","LISTENER3", ...
# A typical install sets only "LISTENER" 
LISTENER_NAMES={"LISTENER"}

#LISTENER_PROTOCOLS;StringList;list of listener addresses (protocols and parameters separated by semicolons)
# The possible values for listener protocols are:
# "TCP;1521","TCPS;2484","NMP;ORAPIPE","IPC;IPCKEY","VI;1521" 
# A typical install sets only "TCP;1521" 
LISTENER_PROTOCOLS={"TCP;1521"}

#LISTENER_START;String;name of the listener to start, in double quotes
LISTENER_START=""LISTENER""

#NAMING_METHODS;StringList;list of naming methods
# The possible values for naming methods are: 
# LDAP, TNSNAMES, ONAMES, HOSTNAME, NOVELL, NIS, DCE
# A typical install sets only: "TNSNAMES","ONAMES","HOSTNAMES" 
# or "LDAP","TNSNAMES","ONAMES","HOSTNAMES" for LDAP
NAMING_METHODS={"TNSNAMES","ONAMES","HOSTNAME"}

#NOVELL_NAMECONTEXT;String;Novell Directory Service name context, in double quotes
# A typical install does not use this variable. 
#NOVELL_NAMECONTEXT = ""NAMCONTEXT""

#SUN_METAMAP;String; SUN meta map, in double quotes
# A typical install does not use this variable. 
#SUN_METAMAP = ""MAP""

#DCE_CELLNAME;String;DCE cell name, in double quotes
# A typical install does not use this variable. 
#DCE_CELLNAME = ""CELL""

#NSN_NUMBER;Number;Number of NetService Names
# A typical install sets one net service name
NSN_NUMBER=1

#NSN_NAMES;StringList;list of Net Service names
# A typical install sets net service name to "EXTPROC_CONNECTION_DATA"
NSN_NAMES={"EXTPROC_CONNECTION_DATA"}

#NSN_SERVICE;StringList;Oracle11g database's service name
# A typical install sets Oracle11g database's service name to "PLSExtProc"
NSN_SERVICE={"PLSExtProc"}

#NSN_PROTOCOLS;StringList;list of coma separated strings of Net Service Name protocol parameters
# The possible values for net service name protocol parameters are:
# "TCP;HOSTNAME;1521","TCPS;HOSTNAME;2484","NMP;COMPUTERNAME;ORAPIPE","VI;HOSTNAME;1521","IPC;IPCKEY"  
# A typical install sets parameters to "IPC;EXTPROC"
NSN_PROTOCOLS={"TCP;HOSTNAME;1521"}
```
</details>


设置完以后，运行netca。
```shell
netca -silent -responsefile /u01/app/oracle/rsp/netca.rsp
```
完成以后检查一下监听状态。
```shell
[oracle@WMS-ZJK-DB rsp]$ lsnrctl status

LSNRCTL for Linux: Version 11.2.0.3.0 - Production on 19-APR-2022 12:23:59

Copyright (c) 1991, 2011, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 11.2.0.3.0 - Production
Start Date                18-APR-2022 19:20:05
Uptime                    0 days 17 hr. 3 min. 54 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/11.2.0/dbhome_1/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/WMS-ZJK-DB/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=WMS-ZJK-DB)(PORT=1521)))
Services Summary...
Service "WMSMDB" has 1 instance(s).
  Instance "WMSMDB", status READY, has 1 handler(s) for this service...
Service "WMSMDBXDB" has 1 instance(s).
  Instance "WMSMDB", status READY, has 1 handler(s) for this service...
The command completed successfully
```
到这一步，基本数据库就安装完毕了。

## 收尾工作

接下来有一些检查项目，强烈建议根据自己的情况执行检查一下。

```shell
# 检查登录是否正常，关键参数是否正确。
sqlplus / as sysdba
show parameter 
# 增大process参数到合理值。
alter system set processes=1000;
# 取消闪回区
alter system set db_recovery_file_dest=''
# 设置归档位置 
alter system set log_archive_dest_1=‘location=/u01/app/oracle/arch’
# 重启数据库到Mount状态，，开启归档
shutdown immediate;
startup mount;
archive log list;
alter database archivelogs;
archive log list;
alter database open;
```

还剩下许多可选的收尾工作，建议逐条跟进一下。
- 建立业务用户和业务表空间；
- 修改密码复杂度和过期时间；
- 设置定时脚本，清理监听，告警日志，归档等；
- 设置RMAN or Expdp 备份相关策略和脚本；
- 考虑修改监听到静态监听；
- 添加监控等等；

# 结语

静默监听，棒；ResponseFile，香。

基本告别图形化界面，linux安装再也不用费劲xclock了。

同时脚本执行也意味着可以做到一键批量部署等进阶运维操作。

图形化安装熟悉后，建议试试rsp安装，摸清rsp文件各个参数也能学到不少。