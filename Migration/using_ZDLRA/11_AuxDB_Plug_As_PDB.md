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


### AUXDB Should Be In READ ONLY Mode

```
# currently AUXDB is a non-container database

# on : ora19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=AUXDB
export ORACLE_SID=AUXDB

$ORACLE_HOME/bin/sqlplus / as sysdba
AUXDB:SYS@SQL>


-- generate XML file which describes the full AUXDB
EXECUTE DBMS_PDB.DESCRIBE(pdb_descr_file => '/tmp/auxdb_to_pdb.xml') ;
show errors ;


-- finally shutdown AUXDB
SHUTDOWN IMMEDIATE ;
exit ;

```


### AUXDB Plug Into O19CPR As PDB

### verify compatibility

```
# on : ora19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CPR
export ORACLE_SID=O19CPR4

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CPR4:SYS@SQL>


-- verify if AUXDB is compatible to become a PDB
set serveroutput on
DECLARE
   compatible BOOLEAN := FALSE ;
BEGIN
   compatible := DBMS_PDB.CHECK_PLUG_COMPATIBILITY(pdb_descr_file => '/tmp/auxdb_to_pdb.xml', pdb_name => 'SALES_PDB') ;

   IF compatible THEN
     DBMS_OUTPUT.PUT_LINE('The Pluggable database is Compatible !!!') ;
   ELSE
     DBMS_OUTPUT.PUT_LINE('ERROR - The Pluggable database is not compatible !!!') ;
   END IF;
END ;
/
show errors ;

-- we are looking for "The Pluggable database is Compatible !!!" output
-- if we get any other output, then we have troubleshoot and figure out how to proceed

```



### AUXDB Plug Into O19CPR As PDB

###### currently AUXDB is a non-container database
###### now we will plug AUXDB as a PDB (Pluggable DataBase) into O19CPR which is a CDB (Container DataBase)

```
# on : ora19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CPR
export ORACLE_SID=O19CPR4

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CPR4:SYS@SQL>


-- we have to use STANDBYS=NONE : because plugging a PDB using XML method will break the STANDBY DB
CREATE PLUGGABLE DATABASE sales_pdb USING '/tmp/auxdb_to_pdb.xml' COPY STANDBYS=NONE ;


-- if a PDB with same name already exists in the CDB, then we can use "AS CLONE" clause
CREATE PLUGGABLE DATABASE sales_pdb_clone AS CLONE USING '/tmp/auxdb_to_pdb.xml' COPY STANDBYS=NONE ;


-- change container to new PDB
ALTER SESSION SET container=SALES_PDB ;
SHOW con_name ;

CON_NAME
------------
SALES_PDB


-- final script to convert non-CDB to PDB
@?/rdbms/admin/noncdb_to_pdb.sql


-- open PDB for READ WRITE
alter pluggable database SALES_PDB open instances=all ;


-- verify if PDB is missing TEMP files or NOT
SELECT file_id, file_name, bytes/(1024*1024*1024) size_gb, autoextensible auto_extend 
FROM dba_temp_files 
WHERE tablespace_name IN 
(SELECT property_value
FROM database_properties
WHERE property_name = 'DEFAULT_TEMP_TABLESPACE') 
;


-- if TEMP files are missing for PDB, then we have to add them
ALTER TABLESPACE TEMP ADD TEMPFILE SIZE 10g ;

```


### SALES_PDB OPEN READ_WRITE OR READ_ONLY

###### now we can OPEN PDB in READ_WRITE OR READ_ONLY mode

```
# on : ora19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CPR
export ORACLE_SID=O19CPR4

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CPR4:SYS@SQL>


-- change container to ROOT CDB
ALTER SESSION SET container=CDB$ROOT ;



-- close PDB
alter pluggable database SALES_PDB close instances=ALL ;

-- open PDB for READ WRITE
alter pluggable database SALES_PDB open instances=ALL ;

-- open PDB for READ ONLY
alter pluggable database SALES_PDB open read only instances=ALL ;

-- save state of PDB for future
alter pluggable database SALES_PDB save state instances=ALL ;

```


### VERIFY ALL PDBs STATUS

```
# on : ora19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CPR
export ORACLE_SID=O19CPR4

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CPR4:SYS@SQL>


-- change container to ROOT CDB
ALTER SESSION SET container=CDB$ROOT ;


SELECT inst_id, con_id, guid, name pdb_name, open_mode, open_time
FROM gv$containers
ORDER BY con_id, inst_id ;

```

---

# APPLICATION DOWNTIME ENDS HERE

```
# on : ora19d.OracleByExample.com : SALES_PDB :


-- Unlock all the application user accounts using :

ALTER USER <app_user> ACCOUNT UNLOCK ;


```

# *** NOW YOU CAN ALLOW YOUR APPLICATION USERS TO CONNECT TO THIS NEW PDB ***

---
