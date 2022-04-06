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
##### we have add tnsnames entries on first node of both PRIMARY and STANDBY clusters

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

###### it is now time to configure STANDBY related settings

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


###### it is now time to restart PRIMARY DB

```
# on : ora19a.OracleByExample.com

$ srvctl status database -db O19CPR
$ srvctl stop   database -db O19CPR
$ srvctl start  database -db O19CPR
$ srvctl status database -db O19CPR
```
