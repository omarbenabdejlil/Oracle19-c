# Install Oracle Database 19c on CentOS 8 By OBA

** Please use the `root` user to edit the files and execute the commands unless further notice. **

## Prerequisite

1. Install the latest [VirtualBox Platform Package](https://www.virtualbox.org/wiki/Downloads) and the [VirtualBox Extension Pack (Oracle_VM_VirtualBox_Extension_Pack-VERSION.vbox-extpack)](https://download.virtualbox.org/virtualbox).
2. Download the latest [VirtualBox Guest Additions (VBoxGuestAdditions_VERSION.iso)](https://download.virtualbox.org/virtualbox).
3. Download the latest [CentOS Stream 8](https://www.centos.org/download/).
4. Create a new virtual machine and install the CentOS to the virtual machine. During the CentOS installation, select `Workstation` as **Base Environment**, select `Container Management`, `Development Tools` and `Graphical Administration Tools` as **Additional software for Selected Environment**. Use `http://mirror.centos.org/centos/8/BaseOS/x86_64/os/` as the installation source.
5. After installing the CentOS, execute the following commands to get the required libraries to create applications for handling compiled objects.

```
dnf update
dnf -y install elfutils-libelf-devel
```

6. Insert the ISO of VirtualBox Guest Additions to the virtual machine, and then install it.

## Download Packages and Software

- [compat-libcap1-1.10-7.el7.x86_64.rpm](https://rpmfind.net/linux/rpm2html/search.php?query=compat-libcap1)
- [compat-libstdc++-33-3.2.3-72.el7.x86_64.rpm](https://rpmfind.net/linux/rpm2html/search.php?query=compat-libstdc%2B%2B)
- [oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm](https://yum.oracle.com/repo/OracleLinux/OL7/latest/x86_64/getPackage/oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm)
- [LINUX.X64_193000_db_home.zip](https://www.oracle.com/database/technologies/oracle-database-software-downloads.html)

## Hostname and Host File

1. Open the file `/etc/hostname`, change the content to update the hostname.

```
ol8-19.localdomain
```

2. Open the file `/etc/hosts`, add your IP address and hostname.

```
192.168.122.1 ol8-19.localdomain
```

## Install Required Packages

1. Perform a dnf update to update every currently installed package.

```bash
dnf update
```

2. Add execute permission to the downloaded rpm files.

```bash
chmod u+x *.rpm
```

3. Install the libcapl library for getting and setting POSIX.1e (formerly POSIX 6) draft 15 capabilities.

```bash
dnf localinstall -y compat-libcap1-1.10-7.el7.x86_64.rpm
```

4. Inatll the libstdc++ package which contains compatibility standard C++ library from GCC 3.3.4.

```bash
dnf localinstall -y compat-libstdc++-33-3.2.3-72.el7.x86_64.rpm
```

5. Install the below required packages.

```bash
dnf install -y bc binutils elfutils-libelf elfutils-libelf-devel fontconfig-devel \
    gcc gcc-c++ glibc glibc-devel ksh ksh libaio libaio-devel libgcc libnsl libnsl.i686 \
    libnsl2 libnsl2.i686 librdmacm-devel libstdc++ libstdc++-devel libX11 libXau libxcb \
    libXi libXrender libXrender-devel libXtst make net-tools nfs-utils smartmontools \
    sysstat targetcli unixODBC;
```

## Install Oracle Installation Prerequisites

1. Install the Oracle Installation Prerequisites (OIP) package.

```bash
dnf localinstall -y oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm
```

2. Open the `/etc/group` file, update the GID of the below items.

```
oinstall:x:64890:oracle
dba:x:64891:oracle
oper:x:64892:oracle
backupdba:x:64893:oracle
dgdba:x:64894:oracle
kmdba:x:64895:oracle
racdba:x:64896:oracle
```

3. Open the `/etc/passwd` file, update both the UID and GID of account `oracle`.

```
oracle:x:64890:64890::/home/oracle:/bin/bash
```

4. Update the password of account `oracle`.

```bash
passwd oracle
```

5. Set secure Linux to permissive by editing the `/etc/selinux/config` file.

```bash
SELINUX=permissive
```

6. Set the secure Linux change right now.

```bash
setenforce Permissive
```

7. Disable the firewall.

```bash
systemctl stop firewalld
systemctl disable firewalld
```

## Setup Oracle User Profile

1. Create Oracle directories.

```bash
mkdir -p /u01/app/oracle/product/19.3.0/dbhome_1
mkdir -p /u02/oradata
chown -R oracle:oinstall /u01 /u02
chmod -R 775 /u01 /u02
```

2. Create a new directory for Oracle user.

```bash
mkdir -p /home/oracle/scripts
chown -R oracle:oinstall /home/oracle
```

3. Create an environment setting file.

```bash
cat > /home/oracle/scripts/setEnv.sh <<EOF
# Oracle Settings
export TMP=/tmp
export TMPDIR=\$TMP

export ORACLE_HOSTNAME=$HOSTNAME
export ORACLE_UNQNAME=cdb1
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=\$ORACLE_BASE/product/19.3.0/dbhome_1
export ORA_INVENTORY=/u01/app/oraInventory
export ORACLE_SID=cdb1
export PDB_NAME=pdb1
export DATA_DIR=/u02/oradata

export PATH=/usr/sbin:/usr/local/bin:\$PATH
export PATH=\$ORACLE_HOME/bin:\$PATH

export LD_LIBRARY_PATH=\$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=\$ORACLE_HOME/jlib:\$ORACLE_HOME/rdbms/jlib
EOF
```

4. Create a startup shell script.

```bash
cat > /home/oracle/scripts/start_all.sh <<EOF
#!/bin/bash
. /home/oracle/scripts/setEnv.sh

export ORAENV_ASK=NO
. oraenv
export ORAENV_ASK=YES

dbstart \$ORACLE_HOME
EOF
```

5. Create a stop shell script.

```bash
cat > /home/oracle/scripts/stop_all.sh <<EOF
#!/bin/bash
. /home/oracle/scripts/setEnv.sh

export ORAENV_ASK=NO
. oraenv
export ORAENV_ASK=YES

dbshut \$ORACLE_HOME
EOF
```

6. Update the owner and permission of the shell scripts and its parent directory.

```bash
chown -R oracle:oinstall /home/oracle
chmod u+x /home/oracle/scripts/*.sh
```

7. Set the environment when the Bash runs whenever it is started interactively.

```bash
cat > /home/oracle/.bashrc <<EOF
#.bashrc

# User specific aliases and functions

alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# Source global definitions
if [ -f /etc/bashrc ]; then
  . /etc/bashrc
fi

. /home/oracle/scripts/setEnv.sh >> /home/oracle/.bashrc
EOF

chown oracle:oinstall /home/oracle/.bashrc
```

## Create and Add New Swap File

1. Run the following command, with **`oracle` user**, to create and apply new swap file.

```bash
dd if=/dev/zero of=/tmp/additional-swap bs=1048576 count=4096
chmod 600 /tmp/additional-swap
mkswap /tmp/additional-swap
```

2. Apply the swap by executing the following command with **`root` user**.

```bash
swapon /tmp/additional-swap
```

## Install Oracle Database

1. Set the DISPLAY variable with **`oracle` user**.

```bash
DISPLAY=$HOSTNAME:0.0; export DISPLAY
```

2. Unzip the archive with **`oracle` user**.

```bash
cd $ORACLE_HOME
unzip -oq /path/to/software/LINUX.X64_193000_db_home.zip
```

3. "Cheat" the installer about the distribution with **`oracle` user**.

```bash
export CV_ASSUME_DISTID=RHEL7.6
```

4. Run the installer, with **`oracle` user**, to install Oracle database.

```bash
cd $ORACLE_HOME
./runInstaller -ignorePrereq -waitforcompletion -silent                        \
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

5. If the setup is success, the following message should be printed on screen.

```
Successfully Setup Software.
```

6. Execute the below scripts, with **`root` user**, to update the permission of Oracle directories and set the environment variables.

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/oracle/product/19.3.0/dbhome_1/root.sh
```

## Database Creation

1. Start the listener with **`oracle` user**.

```bash
lsnrctl start
```

2. Create a database with **`oracle` user**.

```
dbca -silent -createDatabase                                                   \
     -templateName General_Purpose.dbc                                         \
     -gdbname ${ORACLE_SID} -sid  ${ORACLE_SID} -responseFile NO_VALUE         \
     -characterSet AL32UTF8                                                    \
     -sysPassword SysPassword1                                                 \
     -systemPassword SysPassword1                                              \
     -createAsContainerDatabase true                                           \
     -numberOfPDBs 1                                                           \
     -pdbName ${PDB_NAME}                                                      \
     -pdbAdminPassword PdbPassword1                                            \
     -databaseType MULTIPURPOSE                                                \
     -automaticMemoryManagement false                                          \
     -totalMemory 1000                                                         \
     -storageType FS                                                           \
     -datafileDestination "${DATA_DIR}"                                        \
     -redoLogFileSize 50                                                       \
     -emConfiguration NONE                                                     \
     -ignorePreReqs
```

## Listener Update

1. Replace or edit the `listener.ora` file, with **`oracle` user**, to set the correct hostname, port number and SID name.

```bash
cat > /u01/app/oracle/product/19.3.0/dbhome_1/network/admin/listener.ora <<EOF
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(PORT = 1539))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME = ${ORACLE_SID})
    )
  )

EOF
```

2. Reload the Oracle Listener.

```bash 
lsnrctl reload
```

## Post Installation

1. Edit the `/etc/oratab` file, with **`root` user**, to update the restart flag from '`N`' to '`Y`'.

```bash
orcl:/u01/app/oracle/product/19.3.0/dbhome_1:Y
```

2. Configure the Database instance "orcl" with auto startup.

```bash
cd $ORACLE_HOME/dbs
ln -s spfilecdb1.ora initorcl.ora
```

3. Enable Oracle Managed Files (OMF) and make sure the PDB starts when the instance starts.

```bash
sqlplus / as sysdba <<EOF
alter system set db_create_file_dest='${DATA_DIR}';
alter pluggable database ${PDB_NAME} save state;
exit;
EOF
```

4. Execute the following commands, with **`root` user**, to start the Oracle Listener automatically.

```bash
cat > /home/oracle/scripts/cron.sh <<EOF1
#!/bin/bash

. /home/oracle/scripts/setEnv.sh

echo "\`date\`" > /home/oracle/scripts/last.log

lsnrctl start

sleep 3

lsnrctl reload

sleep 3

sqlplus /nolog <<EOF
conn / as sysdba
startup
EOF

EOF1

chown oracle:oinstall /home/oracle/scripts/cron.sh
chmod 744 /home/oracle/scripts/cron.sh
```

5. Use the following command, with **`oracle` user**,  to edit the crontab file.

```bash
crontab -e
```

6. Put the following cron job in the first line of crontab file, then press the keys `:wq` to save and exit.

```bash
@reboot /home/oracle/scripts/cron.sh
```

## Healthcheck

1. Login as **`oracle` user** and then execute the following commands one-by-one.

```bash
sqlplus /nolog
conn / as sysdba;
select * from v$version;
show pdbs;
```

## Create New User and Tablespace

1. Login as Sysdba with SqlPlus.

```bash
sqlplus / as sysdba
```

2. Update the seesion setting `_ORACLE_SCRIPT` to `true` to allow common user comes without `c##` as prefix.

```sql
ALTER SESSION SET "_ORACLE_SCRIPT"=true;
```

3. Create a new tablespace with an automatic extensible size `100MB`, maximum `10G` in size.

```sql
-- DROP TABLESPACE my_tablespace INCLUDING CONTENTS AND DATAFILES;
-- Location of the dat file: /u01/app/oracle/product/19.3.0/dbhome_1/dbs/my_tablespace.dat
-- SELECT tablespace_name, block_size, max_size, status FROM DBA_TABLESPACES;
CREATE TABLESPACE my_tablespace
  DATAFILE 'my_tablespace.dat'
    SIZE 100M
    AUTOEXTEND ON
    NEXT 32M MAXSIZE 10G
    EXTENT MANAGEMENT LOCAL
    SEGMENT SPACE MANAGEMENT AUTO
;

-- List the information of the tablespaces
SELECT FILE_ID, FILE_NAME, TABLESPACE_NAME, AUTOEXTENSIBLE, INCREMENT_BY 
FROM DBA_DATA_FILES ORDER BY FILE_ID DESC;

-- List the free space of the tablespaces
SELECT tablespace_name, SUM(BYTES)/1024/1024 "Free Space (MB)" FROM dba_free_space GROUP BY tablespace_name;

-- Extend the tablespace with the same dat file
-- ALTER DATABASE DATAFILE '/u01/app/oracle/product/19.3.0/dbhome_1/dbs/my_testing_tablespace.dat' RESIZE 20G;
```

4. [Optional] Update the password life time from 180 days (default) to unlimited.
```sql
ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;
```

5. Create a new user.

```sql
-- ALTER SESSION SET "_ORACLE_SCRIPT"=true;
-- DROP USER newuser CASCADE;
CREATE USER newuser IDENTIFIED BY "P@ssw0rd" DEFAULT TABLESPACE my_tablespace;
```

6. Grant permissions to the new user.

```sql
-- REVOKE CREATE SESSION FROM newuser;
-- REVOKE CREATE TABLE FROM newuser;
-- REVOKE CREATE VIEW FROM newuser;
-- REVOKE CREATE ANY TRIGGER FROM newuser;
-- REVOKE CREATE ANY PROCEDURE FROM newuser;
-- REVOKE CREATE SEQUENCE FROM newuser;
-- REVOKE CREATE SYNONYM FROM newuser;
GRANT CREATE SESSION TO newuser;
GRANT CREATE TABLE TO newuser;
GRANT CREATE VIEW TO newuser;
GRANT CREATE ANY TRIGGER TO newuser;
GRANT CREATE ANY PROCEDURE TO newuser;
GRANT CREATE SEQUENCE TO newuser;
GRANT CREATE SYNONYM TO newuser;

ALTER USER newuser QUOTA UNLIMITED ON my_tablespace;
```

7. [Optional] Grant DBA to the new user.

```sql
-- REVOKE DBA FROM newuser;
GRANT DBA TO newuser;
```

## References

- [Install Oracle Database 19c on CentOS 8 in VirtualBox - Github Gist (hkneptune)](https://gist.github.com/hkneptune/d3e80361cf5871dc8840176741ddff50)
- [Running RPM Packages to Install Oracle Database](https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/running-rpm-packages-to-install-oracle-database.html)
- [Oracle Database 19c Installation On Oracle Linux 8 (OL8)](https://oracle-base.com/articles/19c/oracle-db-19c-installation-on-oracle-linux-8)
