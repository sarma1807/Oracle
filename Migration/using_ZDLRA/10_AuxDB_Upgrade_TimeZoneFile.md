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

### AUXDB Restart & Recompile

```
# on : ora19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=AUXDB
export ORACLE_SID=AUXDB

$ORACLE_HOME/bin/sqlplus / as sysdba
AUXDB:SYS@SQL>

SHUTDOWN immediate ;
startup ;


-- recompile all
@?/rdbms/admin/utlrp.sql

```


### AUXDB Restart & Recompile

```
set linesize 200
set pagesize 1000

COLUMN comp_id   FORMAT a15
COLUMN comp_name FORMAT a40
COLUMN version   FORMAT a15
COLUMN status    FORMAT a15

SELECT comp_id, comp_name, version, status FROM dba_registry ORDER BY 1 ;


-- output :

--------------- ---------------------------------------- --------------- ---------------
COMP_ID         COMP_NAME                                VERSION         STATUS
--------------- ---------------------------------------- --------------- ---------------
APEX            Oracle Application Express               3.2.1.00.10     INVALID
CATALOG         Oracle Database Catalog Views            19.0.0.0.0      VALID
CATJAVA         Oracle Database Java Packages            19.0.0.0.0      VALID
CATPROC         Oracle Database Packages and Types       19.0.0.0.0      VALID
JAVAVM          JServer JAVA Virtual Machine             19.0.0.0.0      VALID
ORDIM           Oracle Multimedia                        19.0.0.0.0      VALID
OWM             Oracle Workspace Manager                 19.0.0.0.0      VALID
RAC             Oracle Real Application Clusters         19.0.0.0.0      VALID
XDB             Oracle XML Database                      19.0.0.0.0      VALID
XML             Oracle XDK                               19.0.0.0.0      VALID
--------------- ---------------------------------------- --------------- ---------------

-- NOTICE that all database components are VALID and version for them is 19.0.0.0.0

-- APEX is INVALID and will require dedicated upgrade. We are NOT using APEX, so we will proceed to next step

```

