# SOURCE Cluster

```
# 4 Node RAC running Oracle 11.2.0.4 Grid Infrastructure and RDBMS

ora11a.OracleByExample.com
ora11b.OracleByExample.com
ora11c.OracleByExample.com
ora11d.OracleByExample.com

$ORACLE_BASE=/mnt01/oracle/
$ORACLE_HOME=/mnt01/oracle/product/DBHome11204
$ORACLE_UNQNAME=SALES
```

# TARGET Cluster For Auxiliary DB

```
# 4 Node RAC built using Oracle 19.11.0.0.0 Grid Infrastructure and RDBMS

ora19a.OracleByExample.com
ora19b.OracleByExample.com
ora19c.OracleByExample.com
ora19d.OracleByExample.com

$ORACLE_BASE=/mnt01/oracle/
$ORACLE_HOME=/mnt01/oracle/product/DBHome1911
$ORACLE_UNQNAME=AUXDB
```
---

### Start Auxiliary DB

```
# on : ora19d.OracleByExample.com

$ mkdir -p /mnt01/oracle/admin/AUXDB/adump


$ export ORACLE_BASE=/mnt01/oracle/
$ export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
$ export ORACLE_UNQNAME=AUXDB
$ export ORACLE_SID=AUXDB
$ export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"


$ vi /tmp/AUXDB_pfile.ora
######
### *.compatible='11.2.0.0.0'     <======= this is wrong setting
*.compatible='19.0.0'
*.db_name='AUXDB'
*.db_block_size=8192
*.cluster_database=false
*.control_files='+ASM_FOR_DATA'
*.db_create_file_dest='+ASM_FOR_DATA'
*.db_recovery_file_dest='+ASM_FOR_RECO'
*.db_recovery_file_dest_size=100g
*.diagnostic_dest='/mnt01/oracle/'
*.dispatchers='(PROTOCOL=TCP) (SERVICE=AUXDBXDB)'
*.local_listener='(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=ora19d.OracleByExample.com)(PORT=1521)))'
*.remote_login_passwordfile='EXCLUSIVE'
*.sga_max_size=48g
*.sga_target=48g
*.pga_aggregate_target=12g
*.processes=2000
*.undo_tablespace='UNDOTBS1'
*.audit_file_dest='/mnt01/oracle/admin/AUXDB/adump'
*.audit_trail='db'
######



# start the database instance

$ORACLE_HOME/bin/sqlplus / as sysdba
AUXDB:SYS@SQL>

STARTUP NOMOUNT pfile='/tmp/AUXDB_pfile.ora' ;
ALTER SYSTEM REGISTER ;
exit ;



# alert log file for AUXDB

$ tail -f /mnt01/oracle/diag/rdbms/auxdb/AUXDB/trace/alert_AUXDB.log

```


### ORAPW File For AUXDB

```
# on : ora19d.OracleByExample.com

$ORACLE_HOME/bin/orapwd FILE='/mnt01/oracle/product/DBHome1911/dbs/orapwAUXDB' FORMAT=12.2
Enter password for SYS: <PASSWORD_REMOVED>

```


### static listener for AUXDB

```
# on : ora19d.OracleByExample.com

# as grid user/env settings

export ORACLE_BASE=/mnt01/oracle
export ORACLE_HOME=/mnt01/oracle/grid
export ORACLE_SID=+ASM1


$ vi $ORACLE_HOME/network/admin/listener.ora
###### add following entry
SID_LIST_LISTENER = (SID_LIST = (SID_DESC = (GLOBAL_DBNAME = AUXDB) (ORACLE_HOME = /mnt01/oracle/product/DBHome1911) (SID_NAME = AUXDB)))
######



# reload the listener configuration

$ lsnrctl reload
$ lsnrctl status
$ lsnrctl status | grep -i AUXDB

# output :
$ lsnrctl status | grep -i AUXDB
Service "AUXDB" has 2 instance(s).
  Instance "AUXDB", status UNKNOWN, has 1 handler(s) for this service...
  Instance "AUXDB", status BLOCKED, has 1 handler(s) for this service...
$

### at this point both UNKNOWN & BLOCKED statuses are OK and both are required
```

---

### ORAPW File For AUXDB

```
# on : ora19d.OracleByExample.com

$ORACLE_HOME/bin/orapwd FILE='/mnt01/oracle/product/DBHome1911/dbs/orapwAUXDB' FORMAT=12.2
Enter password for SYS: <PASSWORD_REMOVED>

```

---

### static listener for AUXDB

```
# on : ora19d.OracleByExample.com

# as grid user/env settings

export ORACLE_BASE=/mnt01/oracle
export ORACLE_HOME=/mnt01/oracle/grid
export ORACLE_SID=+ASM1


$ vi $ORACLE_HOME/network/admin/listener.ora
###### add following entry
SID_LIST_LISTENER = (SID_LIST = (SID_DESC = (GLOBAL_DBNAME = AUXDB) (ORACLE_HOME = /mnt01/oracle/product/DBHome1911) (SID_NAME = AUXDB)))
######



# reload the listener configuration

$ lsnrctl reload
$ lsnrctl status
$ lsnrctl status | grep -i AUXDB

# output :
$ lsnrctl status | grep -i AUXDB
Service "AUXDB" has 2 instance(s).
  Instance "AUXDB", status UNKNOWN, has 1 handler(s) for this service...
  Instance "AUXDB", status BLOCKED, has 1 handler(s) for this service...
$

### at this point both UNKNOWN & BLOCKED statuses are OK and both are required
```

---

### tnsnames.ora
##### we have to add tnsnames entries

```
# on : ora11d.OracleByExample.com

# as RDBMS user/env settings

vi $ORACLE_HOME/network/admin/tnsnames.ora
###### add following entries
SALES4 = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = ora11d.OracleByExample.com)(PORT = 1521))(CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = SALES)))
AUXDB  = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = ora19d.OracleByExample.com)(PORT = 1521))(CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = AUXDB)(UR = A)))
######


$ORACLE_HOME/bin/tnsping SALES4 | grep OK
$ORACLE_HOME/bin/tnsping AUXDB  | grep OK


# output from both of above tnsping commands should look like following :
OK (0 msec)
# if we get any other output, then we have to troubleshoot and resolve tns connectivity issues.
```

---

### Connectivity Verification

```
# on : ora11d.OracleByExample.com

# as RDBMS user/env settings

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome11204
export ORACLE_UNQNAME=SALES
export ORACLE_SID=SALES4
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"

$ORACLE_HOME/bin/sqlplus sys@SALES4 as sysdba
$ORACLE_HOME/bin/sqlplus sys@AUXDB  as sysdba

# both of above should work
# if we get any errors, then we have to troubleshoot and resolve connectivity issues.



$ORACLE_HOME/bin/rman target / catalog /@CONNECT_TO_ZDLRA auxiliary sys/ILovePlayStation@AUXDB

# output :
RMAN>
Recovery Manager: Release 11.2.0.4.0 - Production on Fri Mar 21 03:30:00 2022
Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

connected to target database: SALES (DBID=2022032100)
connected to recovery catalog database
connected to auxiliary database: AUXDB (not mounted)
RMAN>

# rman connectivity should work
# if we get any errors, then we have to troubleshoot and resolve connectivity issues.
```

---

### RMAN DUPLICATE 11g DB To AUXDB On 19c CLUSTER
##### we will use LIVE database copy method, but most of the data copy will be done from ZDLRA to 19c CLUSTER

### THIS STEP REQUIRES US TO PRE-CONFIGURE "libra.so" and a "wallet" FOR ZDLRA CREDENTIALS ON SERVER "ora19d.OracleByExample.com"

##### RMAN script

```
# on : ora11d.OracleByExample.com

vi /tmp/AUXDB_dupdb_from_ZDLRA.rmn
######
run {
  # rman will allow upto 64 channels
  # higher number of channels will give higher performance and parallelism and will require higher amount of resource too

  # we have to open auxiliary channels to the sbt lib path and zdlra wallet should point to 19c homes
  ALLOCATE auxiliary CHANNEL sbt01 DEVICE TYPE 'SBT_TAPE' PARMS "SBT_LIBRARY=/mnt01/oracle/product/DBHome1911/lib/libra.so, SBT_PARMS=(RA_WALLET='location=file:/mnt01/oracle/product/DBHome1911/dbs/wallet credential_alias=CONNECT_TO_ZDLRA')" ;
  ALLOCATE auxiliary CHANNEL sbt02 DEVICE TYPE 'SBT_TAPE' PARMS "SBT_LIBRARY=/mnt01/oracle/product/DBHome1911/lib/libra.so, SBT_PARMS=(RA_WALLET='location=file:/mnt01/oracle/product/DBHome1911/dbs/wallet credential_alias=CONNECT_TO_ZDLRA')" ;
  ALLOCATE auxiliary CHANNEL sbt03 DEVICE TYPE 'SBT_TAPE' PARMS "SBT_LIBRARY=/mnt01/oracle/product/DBHome1911/lib/libra.so, SBT_PARMS=(RA_WALLET='location=file:/mnt01/oracle/product/DBHome1911/dbs/wallet credential_alias=CONNECT_TO_ZDLRA')" ;
  ALLOCATE auxiliary CHANNEL sbt04 DEVICE TYPE 'SBT_TAPE' PARMS "SBT_LIBRARY=/mnt01/oracle/product/DBHome1911/lib/libra.so, SBT_PARMS=(RA_WALLET='location=file:/mnt01/oracle/product/DBHome1911/dbs/wallet credential_alias=CONNECT_TO_ZDLRA')" ;

  DUPLICATE TARGET DATABASE TO AUXPOL NOOPEN ;
}
######


vi /tmp/AUXDB_dupdb_from_ZDLRA.sh
######
export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome11204
export ORACLE_SID=SALES4
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"
export RMAN_LOGFILE=/tmp/AUXDB_dupdb_from_ZDLRA_`date +'%Y%m%d-%H%M'`.log
rm $RMAN_LOGFILE

date +'%Y-%m-%d %H:%M' >> $RMAN_LOGFILE
$ORACLE_HOME/bin/rman target / catalog /@CONNECT_TO_ZDLRA auxiliary sys/ILovePlayStation@AUXDB cmdfile /tmp/AUXDB_dupdb_from_ZDLRA.rmn >> $RMAN_LOGFILE
date +'%Y-%m-%d %H:%M' >> $RMAN_LOGFILE
######

```

---

# APPLICATION DOWNTIME STARTS HERE

```
# on : SALES DB running on ora11a.OracleByExample.com :


-- Lock all the application user accounts using :

ALTER USER <app_user> ACCOUNT LOCK ;


-- Kill all the application user sessions which are already connected to the database.


-- perform SALES database using following script which we have configured in the 3rd step : [03_Ora11g_DBBackup_to_ZDLRA.md](https://github.com/sarma1807/Oracle/blob/main/Migration/using_ZDLRA/03_Ora11g_DBBackup_to_ZDLRA.md) :
sh /mnt01/oracle/DBBackup_to_ZDLRA.sh

```

---

#### EXECUTE RMAN DUPLICATE 11g DB TO 19c CLUSTER

```
# on : ora11d.OracleByExample.com

$ sh /tmp/AUXDB_dupdb_from_ZDLRA.sh


# watch the process from log file
$ tail -f /tmp/AUXDB_dupdb_from_ZDLRA_*.log

# when completed without any errors, we should have our SALES 11.2.0.4 version DB duplicated to 19c CLUSTER, but this DB cannot be opened for use.
# next step is to upgrade this DB to 19c using dbupgrade method

```

[SAMPLE_LOG](https://github.com/sarma1807/Oracle/blob/main/Migration/using_ZDLRA/07_AUXDB_dupdb_from_ZDLRA_20220321-0321.log)

---
