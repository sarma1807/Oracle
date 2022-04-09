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


# Container DB

```
# 4 Node RAC built using Oracle 19.11.0.0.0 Grid Infrastructure and RDBMS

ora19a.OracleByExample.com
ora19b.OracleByExample.com
ora19c.OracleByExample.com
ora19d.OracleByExample.com

$ORACLE_BASE=/mnt01/oracle/
$ORACLE_HOME=/mnt01/oracle/product/DBHome1911
$ORACLE_UNQNAME=O19CPR
```

---

### AUXDB Should Be In READ ONLY Mode

```
# on : ora19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=AUXDB
export ORACLE_SID=AUXDB

$ORACLE_HOME/bin/sqlplus / as sysdba
AUXDB:SYS@SQL>


COLUMN inst_id     FORMAT 9999999
COLUMN db_name     FORMAT a10
COLUMN db_role     FORMAT a10
COLUMN open_mode   FORMAT a10
COLUMN host_name   FORMAT a30
COLUMN instance    FORMAT a10

SELECT db.inst_id, db.name db_name, db.database_role db_role, db.open_mode, inst.host_name, inst.instance_name instance,
to_char(startup_time, 'DD-MON-YYYY HH24:MI:SS') startup_time
FROM gv$database db, gv$instance inst
WHERE db.inst_id = inst.inst_id ;

-- output :

 INST_ID DB_NAME    DB_ROLE    OPEN_MODE  HOST_NAME                      INSTANCE   STARTUP_TIME
-------- ---------- ---------- ---------- ------------------------------ ---------- --------------------
       1 AUXDB      PRIMARY    READ ONLY  ora19d.OracleByExample.com     AUXDB      21-MAR-2022 07:31:12



```
