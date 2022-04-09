# Auxiliary DB

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

### AUXDB "Unable to open Spfile" FLOOD CONTROL

```
# in the previous step, we have duplicated SALES DB from Oracle 11g using ZDLRA backups to new server running Oracle 19c

# at this point if we observe our alert log for AUXDB, we will notice a flood of messages like following :

# on : ora19d.OracleByExample.com

$ tail -f /mnt01/oracle/diag/rdbms/auxdb/AUXDB/trace/alert_AUXDB.log

2022-03-21T04:20:25.521835-04:00
ORA-01565: Unable to open Spfile /mnt01/oracle/product/DBHome1911/dbs/spfileAUXPOL.ora.
ORA-01565: Unable to open Spfile /mnt01/oracle/product/DBHome1911/dbs/spfileAUXPOL.ora.
ORA-01565: Unable to open Spfile /mnt01/oracle/product/DBHome1911/dbs/spfileAUXPOL.ora.
ORA-01565: Unable to open Spfile /mnt01/oracle/product/DBHome1911/dbs/spfileAUXPOL.ora.



### FLOOD CONTROL

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=AUXDB
export ORACLE_SID=AUXDB
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"

$ORACLE_HOME/bin/sqlplus / as sysdba
AUXDB:SYS@SQL>

CREATE spfile='/mnt01/oracle/product/DBHome1911/dbs/spfileAUXDB.ora' FROM memory ;

```


### AUXDB Prepare For Upgrade To Oracle 19c (19.11)

```
# on : ora19d.OracleByExample.com

$ORACLE_HOME/bin/sqlplus / as sysdba
AUXDB:SYS@SQL>

SHUTDOWN immediate ;
STARTUP mount ;
ALTER DATABASE open RESETLOGS UPGRADE ;

```


### AUXDB REDO Logfiles

```
# on : ora19d.OracleByExample.com

# by default, AUXDB has very small 50mb REDO Logfiles

$ORACLE_HOME/bin/sqlplus / as sysdba
AUXDB:SYS@SQL>

-- check redo logfile size
SELECT DISTINCT inst_id, (bytes/(1024*1024)) log_size_mb FROM gv$log ORDER BY 1 ;

   INST_ID LOG_SIZE_MB
---------- -----------
         1          50


-- we have 4 redo log groups and 2 redo logfiles in each group
SELECT DISTINCT group#, status, archived FROM gv$log ORDER BY 1 ;

    GROUP# STATUS           ARC
---------- ---------------- ---
         1 UNUSED           YES
         2 UNUSED           YES
         3 CURRENT          YES
         4 UNUSED           YES
         5 UNUSED           YES
         6 UNUSED           YES
         7 UNUSED           YES
         8 UNUSED           YES
         9 UNUSED           YES
        10 UNUSED           YES
        11 UNUSED           YES
        12 UNUSED           YES


-- HAVING SUCH A SMALL SIZE REDO LOGFILES WILL SUFFER DURING FREQUENT LOG SWITCHING/ARCHIVING
-- DBUPGRADE PROCESS WILL TAKE VERY LONG TIME

-- ONE EASY METHOD TO INCREDIBLY SPEED UP THE DBUPGRADE PROCESS, IS TO ADD MANY LARGE REDO LOGFILES

-- we will disable unwanted thread groups
ALTER DATABASE DISABLE thread 2 ;
ALTER DATABASE DISABLE thread 3 ;
ALTER DATABASE DISABLE thread 4 ;


-- we will add 25 redo logfiles - each file of 500mb in size
ALTER DATABASE ADD LOGFILE thread 1 group 101 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 102 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 103 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 104 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 105 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 106 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 107 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 108 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 109 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 110 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 111 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 112 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 113 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 114 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 115 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 116 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 117 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 118 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 119 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 120 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 121 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 122 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 123 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 124 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;
ALTER DATABASE ADD LOGFILE thread 1 group 125 ('+ASM_FOR_DATA', '+ASM_FOR_RECO') SIZE 500m ;


-- we will drop all small 50mb redo logfile groups
ALTER DATABASE DROP logfile group  1 ;
ALTER DATABASE DROP logfile group  2 ;
ALTER DATABASE DROP logfile group  3 ;
ALTER DATABASE DROP logfile group  4 ;
ALTER DATABASE DROP logfile group  5 ;
ALTER DATABASE DROP logfile group  6 ;
ALTER DATABASE DROP logfile group  7 ;
ALTER DATABASE DROP logfile group  8 ;
ALTER DATABASE DROP logfile group  9 ;
ALTER DATABASE DROP logfile group 10 ;
ALTER DATABASE DROP logfile group 11 ;
ALTER DATABASE DROP logfile group 12 ;


-- verify if any small 50mb redo logfile groups still exist
SELECT DISTINCT group#, status, archived FROM gv$log WHERE group# < 100 ORDER BY 1 ;


-- we might have to force couple of log switches and re-attempt to drop logfile groups again
ALTER SYSTEM switch ALL logfile ;


-- after dropping all small redo log groups
-- check redo logfile size
SELECT DISTINCT inst_id, (bytes/(1024*1024)) log_size_mb FROM gv$log ORDER BY 1 ;

   INST_ID LOG_SIZE_MB
---------- -----------
         1         500

```


### AUXDB DISABLE CUSTOM glogin.sql

```
# on : ora19d.OracleByExample.com

# having CUSTOM glogin.sql will cause DBUPGRADE process to fail with UNKNOWN errors

mv $ORACLE_HOME/sqlplus/admin/glogin.sql $ORACLE_HOME/sqlplus/admin/glogin.sql_original

```


### AUXDB Upgrade To Oracle 19c (19.11)

```
# on : ora19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=AUXDB
export ORACLE_SID=AUXDB
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"

mkdir -p /tmp/AUXDB_dbupgrade_logs

# $ORACLE_HOME/bin/dbupgrade -n <parallel jobs> -l <log files location>
$ORACLE_HOME/bin/dbupgrade -n 16 -l /tmp/AUXDB_dbupgrade_logs

```

```
# DBUPGRADE process logs :

/tmp/AUXDB_dbupgrade_logs/catupgrd*.log
/tmp/AUXDB_dbupgrade_logs/upg_summary.log

```

[SAMPLE_OUTPUT](https://github.com/sarma1807/Oracle/blob/main/Migration/using_ZDLRA/09_AuxDB_Upgrade_To_Ora19c_OUTPUT.log)

---
