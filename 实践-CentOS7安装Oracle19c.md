
## 环境

|  OS  |  Kernel |  
|:-----|:--------|
|CentOS 7.6|3.10.0|


#### 安装依赖

```
[root@localhost ~]# yum install bc binutils compat-libcap1 compat-libstdc++ dtrace-modules dtrace-modules-headers dtrace-modules-provider-headers dtrace-utils elfutils-libelf elfutils-libelf elfutils-libelf-devel fontconfig-devel elfutils-libelf-devel-static gcc gcc-c++ glibc glibc-common glibc-devel glibc-headers kernel-headers ksh libaio libaio-devel libgcc libgomp libstdc++ libstdc+±devel libdtrace-ctf-devel libX11 libXau libXi libXtst libXrender libXrender-devel libgcc librdmacm-devel libstdc++ libstdc+±devel libxcb make net-tools nfs-utils python python-configshell python-rtslib python-six smartmontools targetcli unzip vim sysstat unixODBC unixODBC-devel compat-libstdc++-33 xorg-x11-utilsxorg-x11-xauth psmisc xorg-x11-utils xorg-x11-xauth rlwrap
```


#### 扩容swap空间

```
[root@localhost ~]# mkdir /swap
[root@localhost ~]# dd if=/dev/zero of=/swap/swapfile bs=1G count=16
[root@localhost ~]# chmod 600 /swap/swapfile
[root@localhost ~]# mkswap  /swap/swapfile
[root@localhost ~]# swapon  /swap/swapfile
[root@localhost ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:           7821         493         682        3024        6645        4187
Swap:         16383           0       16383
[root@localhost ~]# echo "/swap/swapfile     swap   swap    defaults        0 0 " >> /etc/fstab    //开机自动启用swap
```

#### 安装Oracle19c依赖检查RPM包

```
[root@localhost ~]#  wget -S "https://yum.oracle.com/repo/OracleLinux/OL7/latest/x86_64/getPackage/oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm"
[root@localhost ~]#  rpm -ivh oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm
[root@localhost ~]#  rpm -ql oracle-database-preinstall-19c
/etc/rc.d/init.d/oracle-database-preinstall-19c-firstboot
/etc/security/limits.d/oracle-database-preinstall-19c.conf
/etc/sysconfig/oracle-database-preinstall-19c
/etc/sysconfig/oracle-database-preinstall-19c/oracle-database-preinstall-19c-verify
/etc/sysconfig/oracle-database-preinstall-19c/oracle-database-preinstall-19c.param
/usr/bin/oracle-database-preinstall-19c-verify
/var/log/oracle-database-preinstall-19c
/var/log/oracle-database-preinstall-19c/results
```

#### 设置ORACLE_HOME等环境变量，在oracle用户下
```
[root@localhost ~]# pwd
/home/oracle
[oracle@localhost ~]$ cat .bash_profile
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
	. ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/.local/bin:$HOME/bin

export PATH
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19c/db
export ORACLE_SID=orcl19c
export PATH=$ORACLE_HOME/bin:/usr/sbin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$ORACLE_HOME/lib32:/lib:/usr/lib
```

#### 创建ORACLE_HOME等目录

```
[root@localhost src] mkdir  /u01/app/oracle/oraInventory -pv
[root@localhost src] mkdir -pv  /u01/app/oracle/oradata
[root@localhost src] chown -R oracle:oinstall /u01/
```

#### 拷贝安装包至ORACLE_HOME并解压

```
[root@localhost src]# pwd
/usr/local/src
[root@localhost src]# cp  -rf 19c-package/* /u01/app/oracle/product/19c/db/
```

#### 执行安装前检查

```
[oracle@localhost  db]$  cd $ORACLE_HOME
[oracle@localhost  db]$  ./runInstaller -executePrereqs -silent
```
#### 执行安装

```
[oracle@localhost  db]$ export ORACLE_HOSTNAME=`hostname`
[oracle@localhost  db]$ export ORA_INVENTORY=/u01/app/oracle/oraInventory/

[oracle@localhost  db]$ ./runInstaller -waitforcompletion -silent                                      \
	-responseFile ${ORACLE_HOME}/install/response/db_install.rsp               \
	oracle.install.option=INSTALL_DB_SWONLY                                    \
	ORACLE_HOSTNAME=${ORACLE_HOSTNAME}                                         \
	UNIX_GROUP_NAME=oinstall                                                   \
	INVENTORY_LOCATION=${ORA_INVENTORY}                                        \
	SELECTED_LANGUAGES=en,en_GB                                                \
	ORACLE_HOME=${ORACLE_HOME}                                                 \
	ORACLE_BASE=${ORACLE_BASE}                                                 \
	oracle.install.db.InstallEdition=EE                                        \
	oracle.install.db.OSDBA_GROUP=dba                                          \
	oracle.install.db.OSBACKUPDBA_GROUP=dba                                    \
	oracle.install.db.OSDGDBA_GROUP=dba                                        \
	oracle.install.db.OSKMDBA_GROUP=dba                                        \
	oracle.install.db.OSRACDBA_GROUP=dba                                       \
	SECURITY_UPDATES_VIA_MYORACLESUPPORT=false                                 \
	DECLINE_SECURITY_UPDATES=true
```

#### 创建监听器

```
[oracle@localhost  db]$ netca -silent -responsefile $ORACLE_HOME/assistants/netca/netca.rsp -lisport 1521
```

#### 创建数据库

```
[oracle@localhost  db]$ dbca -silent -createDatabase                                                   \
     -templateName General_Purpose.dbc                                         \
     -gdbname ${ORACLE_SID} -sid  ${ORACLE_SID} -responseFile NO_VALUE         \
     -characterSet AL32UTF8                                                    \
     -sysPassword Oracle_123                                                   \
     -systemPassword Oracle_123                                                \
     -createAsContainerDatabase false                                          \
     -databaseType MULTIPURPOSE                                                \
     -automaticMemoryManagement false                                          \
     -totalMemory 1000                                                         \
     -storageType FS                                                           \
     -datafileDestination " /u01/app/oracle/oradata"                           \
     -redoLogFileSize 50                                                       \
     -emConfiguration NONE                                                     \
     -ignorePreReqs
```

#### 验证

```
[oracle@localhost  db]$  lsnrctl  status

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 13-APR-2021 14:57:36

Copyright (c) 1991, 2019, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=oracledgtest-shylf-12)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Start Date                12-APR-2021 14:42:31
Uptime                    1 days 0 hr. 15 min. 4 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/19c/db/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/oracledgtest-shylf-12/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oracledgtest-shylf-12)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
Services Summary...
Service "orcl19c" has 1 instance(s).
  Instance "orcl19c", status READY, has 1 handler(s) for this service...
Service "orcl19cXDB" has 1 instance(s).
  Instance "orcl19c", status READY, has 1 handler(s) for this service...
The command completed successfully
[oracle@localhost  db]$ 
```

 