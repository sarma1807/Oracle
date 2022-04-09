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
