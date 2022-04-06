```
Creating a Physical Standby using RMAN Duplicate (RAC or Non-RAC) (Oracle Support Doc ID 1617946.1)
Step by Step Guide on Creating Physical Standby Using RMAN DUPLICATE...FROM ACTIVE DATABASE (Oracle Support Doc ID 1075908.1)
```

# PRIMARY Cluster

```
# 4 Node RAC built using Oracle 19.11.0.0.0 Grid Infrastructure and RDBMS

ora19a.OracleByExample.com
ora19b.OracleByExample.com
ora19c.OracleByExample.com
ora19d.OracleByExample.com

$ORACLE_BASE=/mnt01/oracle/
$ORACLE_HOME=/mnt01/oracle/product/DBHome1911
```


# STANDBY Cluster

```
# 4 Node RAC built using Oracle 19.11.0.0.0 Grid Infrastructure and RDBMS

odr19a.OracleByExample.com
odr19b.OracleByExample.com
odr19c.OracleByExample.com
odr19d.OracleByExample.com

$ORACLE_BASE=/mnt01/oracle/
$ORACLE_HOME=/mnt01/oracle/product/DBHome1911
```

---

### DISABLE Data Guard Broker
##### we will NOT configure/use Data Guard Broker for this cluster

```
# on : ora19a.OracleByExample.com

O19CPR1:SYS@SQL>

SHOW PARAMETER dg

NAME                    TYPE     VALUE
----------------------- -------- ------------------------------
dg_broker_config_file1  string   /mnt01/oracle/product/DBHome1911/dbs/dr1O19CPR.dat
dg_broker_config_file2  string   /mnt01/oracle/product/DBHome1911/dbs/dr2O19CPR.dat
dg_broker_start         boolean  FALSE


ALTER SYSTEM SET dg_broker_config_file1='+ASM_FOR_DATA/O19CPR/dr1O19CPR.dat' SCOPE=both SID='*' ;
ALTER SYSTEM SET dg_broker_config_file2='+ASM_FOR_DATA/O19CPR/dr2O19CPR.dat' SCOPE=both SID='*' ;
ALTER SYSTEM SET dg_broker_start=FALSE                                       SCOPE=both SID='*' ;
```

---

### Copy ORAPW File
##### we have to copy ORAPW (password) file from PRIMARY to STANDBY

```
# on : ora19a.OracleByExample.com

# as grid user/env settings

export ORACLE_BASE=/mnt01/oracle
export ORACLE_HOME=/mnt01/oracle/grid
export ORACLE_SID=+ASM1


# get ORAPW file details from PRIMARY DB
$ORACLE_HOME/bin/srvctl config database -db O19CPR | egrep "name|Password"
Database unique name: O19CPR
Database name: O19CPR
Password file: +ASM_FOR_DATA/O19CPR/PASSWORD/pwdO19CPR.2002.2002032100
$


# copy ORAPW file from ASM to FS
$ORACLE_HOME/bin/asmcmd cp +ASM_FOR_DATA/O19CPR/PASSWORD/pwdO19CPR.2002.2002032100 /tmp/orapwO19CPR1


# scp ORAPW file to first node on STANDBY cluster
$ scp /tmp/orapwO19CPR1 oracle@odr19a.OracleByExample.com:/mnt01/oracle/product/DBHome1911/dbs/orapwO19CDR1
```

---

### tnsnames.ora
##### we have to add tnsnames entries on first node of both PRIMARY and STANDBY clusters

```
# on : ora19a.OracleByExample.com & odr19a.OracleByExample.com

# as RDBMS user/env settings

vi $ORACLE_HOME/network/admin/tnsnames.ora
###### add following entries
O19CPR = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = ora19a.OracleByExample.com)(PORT = 1521))(CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = O19CPR)))
O19CDR = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = odr19a.OracleByExample.com)(PORT = 1521))(CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = O19CDR)(UR = A)))
######


$ORACLE_HOME/bin/tnsping O19CPR | grep OK
$ORACLE_HOME/bin/tnsping O19CDR | grep OK


# output from both of above tnsping commands should look like following :
OK (0 msec)
# if we get any other output, then we have to troubleshoot and resolve tns connectivity issues.
```

---

### PRIMARY DB verification & configuration

```
# on : ora19a.OracleByExample.com

O19CPR1:SYS@SQL>

-- make sure PRIMARY DB is in archive log mode

archive log list

-- output :
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            +ASM_FOR_RECO
Oldest online log sequence     2002
Next log sequence to archive   2005
Current log sequence           2005
SQL>


-- force logging must be enabled

SELECT force_logging FROM v$database ;

-- output :
FORCE_LOGGING
---------------
YES


-- if force logging needs to be enabled, use following command :

SQL> ALTER DATABASE force logging ;



-- flashback must be enabled

SELECT flashback_on FROM v$database ;

-- output :
FLASHBACK_ON
---------------
YES

-- if flashback needs to be enabled, use following command :

SQL> ALTER DATABASE flashback on ;

```

##### it is now time to configure STANDBY related settings

```
# on : ora19a.OracleByExample.com

O19CPR1:SYS@SQL>

alter system set LOG_ARCHIVE_CONFIG='DG_CONFIG=(O19CPR,O19CDR)' SID='*' SCOPE=SPFILE ;
alter system set LOG_ARCHIVE_DEST_1='LOCATION=+ASM_FOR_RECO VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=O19CPR' SID='*' SCOPE=SPFILE ;
alter system set LOG_ARCHIVE_DEST_2='SERVICE=O19CDR LGWR ASYNC VALID_FOR=(ONLINE_LOGFILE,PRIMARY_ROLE) DB_UNIQUE_NAME=O19CDR' SID='*' SCOPE=SPFILE ;
alter system set LOG_ARCHIVE_DEST_STATE_1=ENABLE SID='*' SCOPE=SPFILE ;
alter system set LOG_ARCHIVE_DEST_STATE_2=ENABLE SID='*' SCOPE=SPFILE ;
-- server is STANDBY and client is PRIMARY
alter system set FAL_SERVER=O19CDR SID='*' SCOPE=SPFILE ;
alter system set FAL_CLIENT=O19CPR SID='*' SCOPE=SPFILE ;
alter system set STANDBY_FILE_MANAGEMENT=auto SID='*' SCOPE=SPFILE ;

-- on PRIMARY : standby path, primary path, standby path, primary path
-- on STANDBY : primary path, standby path, primary path, standby path

alter system set  DB_FILE_NAME_CONVERT='+ASM_FOR_DATA/O19CDR/','+ASM_FOR_DATA/O19CPR/','+ASM_FOR_RECO/O19CDR/','+ASM_FOR_RECO/O19CPR/' SID='*' SCOPE=SPFILE ;
alter system set LOG_FILE_NAME_CONVERT='+ASM_FOR_DATA/O19CDR/','+ASM_FOR_DATA/O19CPR/','+ASM_FOR_RECO/O19CDR/','+ASM_FOR_RECO/O19CPR/' SID='*' SCOPE=SPFILE ;
alter system set PDB_FILE_NAME_CONVERT='+ASM_FOR_DATA/O19CDR/','+ASM_FOR_DATA/O19CPR/','+ASM_FOR_RECO/O19CDR/','+ASM_FOR_RECO/O19CPR/' SID='*' SCOPE=SPFILE ;
```


##### it is now time to restart PRIMARY DB

```
# on : ora19a.OracleByExample.com

$ srvctl status database -db O19CPR
$ srvctl stop   database -db O19CPR
$ srvctl start  database -db O19CPR
$ srvctl status database -db O19CPR
```

---

### STANDBY DB
##### we will create a basic pfile and startup nomount STANDBY DB instance on first node

###### required folders

```
# on all 4 nodes of STANDBY cluster : odr19a.OracleByExample.com/odr19b.OracleByExample.com/odr19c.OracleByExample.com/odr19d.OracleByExample.com

$ mkdir -p /mnt01/oracle/admin/O19CDR/adump
$ mkdir -p /mnt01/oracle/admin/O19CDR/dpdump
$ mkdir -p /mnt01/oracle/admin/O19CDR/hdump
$ mkdir -p /mnt01/oracle/admin/O19CDR/pfile
$ mkdir -p /mnt01/oracle/admin/O19CDR/xdb_wallet
```


###### STANDBY DB pfile

```
# on : odr19a.OracleByExample.com

$ vi /tmp/initO19CDR.ora
###
DB_NAME=O19CDR
DB_UNIQUE_NAME=O19CDR
DB_BLOCK_SIZE=8192
SGA_TARGET=16G
PGA_AGGREGATE_TARGET=4G
LOCAL_LISTENER='(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=odr19a.OracleByExample.com)(PORT=1521)))'
ENABLE_PLUGGABLE_DATABASE=TRUE
PROCESSES=3000
###
```


###### STANDBY DB startup

```
# on : odr19a.OracleByExample.com

# as RDBMS user/env settings

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR1
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR1:SYS@SQL>

STARTUP NOMOUNT pfile='/tmp/initO19CDR.ora' ;
ALTER SYSTEM register ;

```


###### STANDBY DB alert log file

```
tail -f /mnt01/oracle/diag/rdbms/o19cdr/O19CDR?/trace/alert_O19CDR?.log
```


### static listener for STANDBY DB

```
# on : odr19a.OracleByExample.com

# as grid user/env settings

export ORACLE_BASE=/mnt01/oracle
export ORACLE_HOME=/mnt01/oracle/grid
export ORACLE_SID=+ASM1


$ vi $ORACLE_HOME/network/admin/listener.ora
###### add following entry
SID_LIST_LISTENER = (SID_LIST = (SID_DESC = (GLOBAL_DBNAME = O19CDR) (ORACLE_HOME = /mnt01/oracle/product/DBHome1911) (SID_NAME = O19CDR1)))
######



# reload the listener configuration

$ lsnrctl reload
$ lsnrctl status
$ lsnrctl status | grep -i O19CDR

# output :
$ lsnrctl status | grep -i O19CDR
Service "O19CDR" has 2 instance(s).
  Instance "O19CDR1", status UNKNOWN, has 1 handler(s) for this service...
  Instance "O19CDR1", status BLOCKED, has 1 handler(s) for this service...
$

### at this point both UNKNOWN & BLOCKED statuses are OK and both are required
```

---

### Connectivity Verification

```
# on : ora19a.OracleByExample.com and odr19a.OracleByExample.com

# as RDBMS user/env settings

$ORACLE_HOME/bin/sqlplus sys@O19CPR as sysdba
$ORACLE_HOME/bin/sqlplus sys@O19CDR as sysdba

# both of above should work
# if we get any errors, then we have to troubleshoot and resolve connectivity issues.
```

```
# on : odr19a.OracleByExample.com

# as RDBMS user/env settings

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR1
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"

$ORACLE_HOME/bin/rman target sys@O19CPR auxiliary /


# output :
RMAN>
Recovery Manager: Release 19.0.0.0.0 - Production on Fri Mar 21 03:30:00 2022
Version 19.11.0.0.0
Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.
target database Password: <PASSWORD_REMOVED>
connected to target database: O19CPR (DBID=2002032100)
connected to auxiliary database: O19CDR (not mounted)
RMAN>

# rman connectivity should work
# if we get any errors, then we have to troubleshoot and resolve connectivity issues.
```

---

### SRVCTL ADD DATABASE

```
# on : odr19a.OracleByExample.com

# as grid user/env settings

export ORACLE_BASE=/mnt01/oracle
export ORACLE_HOME=/mnt01/oracle/grid
export ORACLE_SID=+ASM1

# establish the asm diskgroup dependency for the future standby db

$ORACLE_HOME/bin/srvctl add database -db O19CDR -oraclehome /mnt01/oracle/product/DBHome1911 -diskgroup "ASM_FOR_DATA,ASM_FOR_RECO"


$ORACLE_HOME/bin/srvctl status database -db O19CDR
# output :
Database is not running.



$ORACLE_HOME/bin/crsctl status resource ora.o19cdr.db -t

# output :
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.o19cdr.db
      1        OFFLINE OFFLINE                               STABLE
      2        OFFLINE OFFLINE                               STABLE
      3        OFFLINE OFFLINE                               STABLE
      4        OFFLINE OFFLINE                               STABLE
--------------------------------------------------------------------------------
```

---

### RMAN DUPLICATE PRIMARY DATABASE
##### we will use LIVE database copy method, it might impact applications if PRIMARY DB is under heavy load


###### RMAN script

```
# on : odr19a.OracleByExample.com


$ vi /tmp/O19CDR_dupdb_from_primary.rmn
######
run {
  # rman will allow upto 64 channels
  # higher number of channels will give higher performance and parallelism and will require higher amount of resource too

  allocate channel primary_01 type disk ;
  allocate channel primary_02 type disk ;
  allocate channel primary_03 type disk ;
  allocate channel primary_04 type disk ;

  allocate auxiliary channel standby_01 type disk ;
  allocate auxiliary channel standby_02 type disk ;
  allocate auxiliary channel standby_03 type disk ;
  allocate auxiliary channel standby_04 type disk ;

  DUPLICATE TARGET DATABASE for standby FROM active database DORECOVER NOFILENAMECHECK ;
}
######

```


###### RMAN shell script

```
# on : odr19a.OracleByExample.com


$ vi /tmp/O19CDR_dupdb_from_primary.sh
######
export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR1
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"
export RMAN_LOGFILE=/tmp/O19CDR_dupdb_from_primary_`date +'%Y%m%d-%H%M'`.log
rm $RMAN_LOGFILE

date +'%Y-%m-%d %H:%M' >> $RMAN_LOGFILE
$ORACLE_HOME/bin/rman target sys/ILovePlayStation@O19CPR auxiliary sys/ILovePlayStation@O19CDR cmdfile /tmp/O19CDR_dupdb_from_primary.rmn >> $RMAN_LOGFILE
date +'%Y-%m-%d %H:%M' >> $RMAN_LOGFILE
######

```


#### EXECUTE RMAN DUPLICATE PRIMARY DATABASE TO STANDBY

```
# on : odr19a.OracleByExample.com

$ sh /tmp/O19CDR_dupdb_from_primary.sh


# watch the process from log file
$ tail -f /tmp/O19CDR_dupdb_from_primary_*.log

# when completed without any errors, we should have our standby database ready for next steps

```

---

### FINALIZE STANDBY DATABASE


###### standby database configuration changes

```
# on : odr19a.OracleByExample.com

# as RDBMS user/env settings

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR1

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR1:SYS@SQL>

-- standby related settings
alter system set LOG_ARCHIVE_CONFIG='DG_CONFIG=(O19CDR,O19CPR)' ;
alter system set LOG_ARCHIVE_DEST_1='LOCATION=+ASM_FOR_RECO VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=O19CDR' ;
alter system set LOG_ARCHIVE_DEST_2='SERVICE=O19CPR LGWR ASYNC VALID_FOR=(ONLINE_LOGFILE,PRIMARY_ROLE) DB_UNIQUE_NAME=O19CPR' ;
alter system set LOG_ARCHIVE_DEST_STATE_1=ENABLE ;
alter system set FAL_SERVER=O19CPR ;
alter system set FAL_CLIENT=O19CDR ;

-- asm disk group related settings
alter system set DB_RECOVERY_FILE_DEST_SIZE=200G ;
alter system set DB_CREATE_FILE_DEST='+ASM_FOR_DATA' ;
alter system set DB_RECOVERY_FILE_DEST='+ASM_FOR_RECO' ;
alter system set DB_CREATE_ONLINE_LOG_DEST_1='+ASM_FOR_DATA' ;
alter system set DB_CREATE_ONLINE_LOG_DEST_2='+ASM_FOR_RECO' ;
```


###### standby database create SPFILE

```
# on : odr19a.OracleByExample.com

# as RDBMS user/env settings

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR1

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR1:SYS@SQL>

-- finally create a spfile
CREATE spfile='+ASM_FOR_DATA' FROM memory ;
```


###### PRIMARY DB extra settings

```
# on : ora19a.OracleByExample.com

# as RDBMS user/env settings

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CPR
export ORACLE_SID=O19CPR1

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CPR1:SYS@SQL>

-- asm disk group related settings
alter system set DB_RECOVERY_FILE_DEST_SIZE=200G             sid='*' scope=BOTH ;
alter system set DB_CREATE_FILE_DEST='+ASM_FOR_DATA'         sid='*' scope=BOTH ;
alter system set DB_RECOVERY_FILE_DEST='+ASM_FOR_RECO'       sid='*' scope=BOTH ;
alter system set DB_CREATE_ONLINE_LOG_DEST_1='+ASM_FOR_DATA' sid='*' scope=BOTH ;
alter system set DB_CREATE_ONLINE_LOG_DEST_2='+ASM_FOR_RECO' sid='*' scope=BOTH ;
```

---

### CONVERT STANDBY AS RAC DATABASE 

###### shutdown standby database

```
# on : odr19a.OracleByExample.com

# as RDBMS user/env settings

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR1

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR1:SYS@SQL>

shutdown immediate ;
```

###### SRVCTL STANDBY DATABASE

```
# on : odr19a.OracleByExample.com

# as grid user/env settings

export ORACLE_BASE=/mnt01/oracle
export ORACLE_HOME=/mnt01/oracle/grid
export ORACLE_SID=+ASM1

### NOTE : ORACLE ACTIVE DATA GUARD WILL REQUIRE SPECIAL LICENSES

$ORACLE_HOME/bin/srvctl modify database -db O19CDR -s 'READ ONLY'
$ORACLE_HOME/bin/srvctl modify database -db O19CDR -role PHYSICAL_STANDBY


### add all 4 instances

$ORACLE_HOME/bin/srvctl add instance -db O19CDR -instance O19CDR1 -node odr19a
$ORACLE_HOME/bin/srvctl add instance -db O19CDR -instance O19CDR2 -node odr19b
$ORACLE_HOME/bin/srvctl add instance -db O19CDR -instance O19CDR3 -node odr19c
$ORACLE_HOME/bin/srvctl add instance -db O19CDR -instance O19CDR4 -node odr19d


### ORAPW settings

$ cp /mnt01/oracle/product/DBHome1911/dbs/orapwO19CDR1 /tmp/orapwO19CDR

$ORACLE_HOME/bin/asmcmd pwget    --dbuniquename O19CDR
$ORACLE_HOME/bin/asmcmd pwdelete --dbuniquename O19CDR
# at this point ORAPW file should NOT exist on ASM Disk Group

$ORACLE_HOME/bin/asmcmd mkdir +ASM_FOR_DATA/O19CDR/PASSWORD/
$ORACLE_HOME/bin/asmcmd pwcopy --dbuniquename O19CDR /tmp/orapwO19CDR +ASM_FOR_DATA/O19CDR/PASSWORD/

$ORACLE_HOME/bin/asmcmd pwget --dbuniquename O19CDR
$ORACLE_HOME/bin/asmcmd ls -l +ASM_FOR_DATA/O19CDR/PASSWORD/
# at this point ORAPW file should exist on ASM Disk Group


$ORACLE_HOME/bin/srvctl config database -db O19CDR | grep Password
# at this point this should now show current ORAPW file and should properly be pointing to ASM


# following files are no longer required
$ rm /mnt01/oracle/product/DBHome1911/dbs/orapwO19CDR1
$ rm /tmp/orapwO19CDR
```

```
$ORACLE_HOME/bin/srvctl config database -db O19CDR

# output :
$
Database unique name: O19CDR
Database name:
Oracle home: /mnt01/oracle/product/DBHome1911
Oracle user: oracle
Spfile: +ASM_FOR_DATA/O19CDR/PARAMETERFILE/spfile.202203.2002032100
Password file: +ASM_FOR_DATA/O19CDR/PASSWORD/orapwO19CDR
Domain:
Start options: read only
Stop options: immediate
Database role: PHYSICAL_STANDBY
Management policy: AUTOMATIC
Server pools:
Disk Groups: ASM_FOR_DATA,ASM_FOR_RECO
Mount point paths:
Services:
Type: RAC
Start concurrency:
Stop concurrency:
OSDBA group: dba
OSOPER group: dba
Database instances: O19CDR1,O19CDR2,O19CDR3,O19CDR4
Configured nodes: odr19a,odr19b,odr19c,odr19d
CSS critical: no
CPU count: 0
Memory target: 0
Maximum memory: 0
Default network number for database services:
Database is administrator managed
$
```

---

### STANDBY DB : MOVE CONTROLFILE TO ASM
###### standby db controlfile is still on filesystem, we need to move it to asm and update the spfile

```
# on : odr19a.OracleByExample.com

# as RDBMS user/env settings

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR1

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR1:SYS@SQL>

shutdown immediate ;

startup nomount ;
show parameter control_files ;

-- output :
NAME           TYPE    VALUE
-------------- ------- ------------------------------
control_files  string  /mnt01/oracle/product/DBHome1911/dbs/cntrlO19CDR1.dbf
SQL> exit ;



$ORACLE_HOME/bin/rman target /
RMAN>

RESTORE CONTROLFILE TO '+ASM_FOR_DATA' from '/mnt01/oracle/product/DBHome1911/dbs/cntrlO19CDR1.dbf' ;
RESTORE CONTROLFILE TO '+ASM_FOR_RECO' from '/mnt01/oracle/product/DBHome1911/dbs/cntrlO19CDR1.dbf' ;
RMAN> exit ;



$ORACLE_HOME/bin/asmcmd ls -l +ASM_FOR_DATA/O19CDR/CONTROLFILE/
$ORACLE_HOME/bin/asmcmd ls -l +ASM_FOR_RECO/O19CDR/CONTROLFILE/

# from output of above commands we can see that controlfiles are in following locations :

+ASM_FOR_DATA/O19CDR/CONTROLFILE/current.202203.2002032100
+ASM_FOR_RECO/O19CDR/CONTROLFILE/current.200203.2022032100



$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR1:SYS@SQL>

ALTER SYSTEM SET control_files='+ASM_FOR_DATA/O19CDR/CONTROLFILE/current.202203.2002032100','+ASM_FOR_RECO/O19CDR/CONTROLFILE/current.200203.2022032100' SCOPE=spfile SID='*' ;

ALTER SYSTEM SET cluster_database=TRUE SCOPE=spfile SID='*' ;
ALTER SYSTEM SET cluster_database_instances=4 SCOPE=spfile SID='*' ;

ALTER SYSTEM SET instance_number=1 SCOPE=spfile SID='O19CDR1' ;
ALTER SYSTEM SET instance_number=2 SCOPE=spfile SID='O19CDR2' ;
ALTER SYSTEM SET instance_number=3 SCOPE=spfile SID='O19CDR3' ;
ALTER SYSTEM SET instance_number=4 SCOPE=spfile SID='O19CDR4' ;

ALTER SYSTEM SET thread=1 SCOPE=spfile SID='O19CDR1' ;
ALTER SYSTEM SET thread=2 SCOPE=spfile SID='O19CDR2' ;
ALTER SYSTEM SET thread=3 SCOPE=spfile SID='O19CDR3' ;
ALTER SYSTEM SET thread=4 SCOPE=spfile SID='O19CDR4' ;

ALTER SYSTEM SET undo_tablespace='UNDOTBS1' SCOPE=spfile SID='O19CDR1' ;
ALTER SYSTEM SET undo_tablespace='UNDOTBS2' SCOPE=spfile SID='O19CDR2' ;
ALTER SYSTEM SET undo_tablespace='UNDOTBS3' SCOPE=spfile SID='O19CDR3' ;
ALTER SYSTEM SET undo_tablespace='UNDOTBS4' SCOPE=spfile SID='O19CDR4' ;


ALTER SYSTEM SET local_listener='(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=odr19a-vip.OracleByExample.com)(PORT=1521)))' SCOPE=spfile SID='O19CDR1' ;
ALTER SYSTEM SET local_listener='(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=odr19b-vip.OracleByExample.com)(PORT=1521)))' SCOPE=spfile SID='O19CDR2' ;
ALTER SYSTEM SET local_listener='(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=odr19c-vip.OracleByExample.com)(PORT=1521)))' SCOPE=spfile SID='O19CDR3' ;
ALTER SYSTEM SET local_listener='(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=odr19d-vip.OracleByExample.com)(PORT=1521)))' SCOPE=spfile SID='O19CDR4' ;

alter system set  DB_FILE_NAME_CONVERT='+ASM_FOR_DATA/O19CPR/','+ASM_FOR_DATA/O19CDR/','+ASM_FOR_RECO/O19CPR/','+ASM_FOR_RECO/O19CDR/' SID='*' SCOPE=spfile ;
alter system set LOG_FILE_NAME_CONVERT='+ASM_FOR_DATA/O19CPR/','+ASM_FOR_DATA/O19CDR/','+ASM_FOR_RECO/O19CPR/','+ASM_FOR_RECO/O19CDR/' SID='*' SCOPE=spfile ;
alter system set PDB_FILE_NAME_CONVERT='+ASM_FOR_DATA/O19CPR/','+ASM_FOR_DATA/O19CDR/','+ASM_FOR_RECO/O19CPR/','+ASM_FOR_RECO/O19CDR/' SID='*' SCOPE=spfile ;

alter system set STANDBY_FILE_MANAGEMENT=auto SCOPE=spfile SID='*' ;

ALTER SYSTEM SET audit_file_dest='/mnt01/oracle/admin/O19CDR/adump' SCOPE=spfile SID='*' ;
ALTER SYSTEM SET diagnostic_dest='/mnt01/oracle' SCOPE=both SID='*' ;

SQL> exit ;
```

---

### ORATAB ENTRIES

```
# on all 4 nodes of STANDBY cluster : odr19a.OracleByExample.com/odr19b.OracleByExample.com/odr19c.OracleByExample.com/odr19d.OracleByExample.com

$ vi /etc/oratab
### need to add following entry, it is it not already added
O19CDR:/mnt01/oracle/product/DBHome1911:N
###


$ cat /etc/oratab | grep -i O19CDR
```

---

### STARTUP STANDBY DATABASE

```
# on : odr19a.OracleByExample.com

# as RDBMS user/env settings

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR1

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR1:SYS@SQL>


-- at this point on standby only 1 instance is running and that is on node 1

shutdown immediate ;
exit ;


$ORACLE_HOME/bin/srvctl start  database -db O19CDR
$ORACLE_HOME/bin/srvctl status database -db O19CDR

# output :
$
Instance O19CDR1 is running on node odr19a
Instance O19CDR2 is running on node odr19b
Instance O19CDR3 is running on node odr19c
Instance O19CDR4 is running on node odr19d
$
```

---

### VERIFY FILE_NAME_CONVERT and STANDBY_FILE_MANAGEMENT

```
-- on both PRIMARY and STANDBY databases

SQL> show parameter FILE_NAME_CONVERT ;

-- make sure these are properly configured



SQL> show parameter STANDBY_FILE_MANAGEMENT

-- make sure this is set to AUTO - not MANUAL
```

---

### STARTUP STANDBY DATABASE

```
# on : odr19a.OracleByExample.com

# as RDBMS user/env settings

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR1

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR1:SYS@SQL>


-- start MRP process on standby to apply redo logs from primary
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE cancel ;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE disconnect from session ;
```

---

### tnsnames.ora
##### we have to add tnsnames entries on ALL nodes of both PRIMARY and STANDBY clusters
##### make sure to use SCAN NAME instead of pointing them to single nodes

```
# as RDBMS user/env settings

vi $ORACLE_HOME/network/admin/tnsnames.ora
###### add following entries
O19CPR = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = ora19-scan.OracleByExample.com)(PORT = 1800))(CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = O19CPR)))
O19CDR = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = odr19-scan.OracleByExample.com)(PORT = 1800))(CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = O19CDR)))
######



# we have verify these from/on ALL nodes of both PRIMARY and STANDBY clusters
$ORACLE_HOME/bin/tnsping O19CPR | egrep -i "OK|SERVICE_NAME"
$ORACLE_HOME/bin/tnsping O19CDR | egrep -i "OK|SERVICE_NAME"


# output from both of above tnsping commands should look good and they should be pointing to SCAN NAME.
# if we get any errors, then we have to troubleshoot and resolve connectivity issues.
```

---

### REMOVE static listener for STANDBY DB

```
# on : odr19a.OracleByExample.com

# as grid user/env settings

export ORACLE_BASE=/mnt01/oracle
export ORACLE_HOME=/mnt01/oracle/grid
export ORACLE_SID=+ASM1


$ vi $ORACLE_HOME/network/admin/listener.ora
###### REMOVE following entry - this entry is no longer required
# SID_LIST_LISTENER = (SID_LIST = (SID_DESC = (GLOBAL_DBNAME = O19CDR) (ORACLE_HOME = /mnt01/oracle/product/DBHome1911) (SID_NAME = O19CDR1)))
######



# reload the listener configuration

$ lsnrctl reload
$ lsnrctl status
$ lsnrctl status | grep -i O19CDR

# output :
$ lsnrctl status | grep -i O19CDR
Service "O19CDR" has 1 instance(s).
  Instance "O19CDR1", status READY, has 1 handler(s) for this service...
$

### at this point we should have only READY status
```

---

## DATA GUARD : PROTECTION_MODE/PROTECTION_LEVEL

```
SELECT inst_id, protection_mode, protection_level FROM gv$database ORDER BY 1 ;

   INST_ID PROTECTION_MODE      PROTECTION_LEVEL
---------- -------------------- --------------------
         1 MAXIMUM PERFORMANCE  MAXIMUM PERFORMANCE
         2 MAXIMUM PERFORMANCE  MAXIMUM PERFORMANCE
         3 MAXIMUM PERFORMANCE  MAXIMUM PERFORMANCE
         4 MAXIMUM PERFORMANCE  MAXIMUM PERFORMANCE


-- PROTECTION_MODE/PROTECTION_LEVEL : most suitable option is to set it to MAXIMUM PERFORMANCE


-- if this needs to be changed :
ALTER DATABASE SET STANDBY DATABASE TO maximize performance ;

```

---
