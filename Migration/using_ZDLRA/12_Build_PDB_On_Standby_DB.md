```
Making Use Deferred PDB Recovery and the STANDBYS=NONE Feature with Oracle Multitenant (Oracle Support Doc ID 1916648.1)
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
$ORACLE_UNQNAME=O19CPR
PDB_NAME=SALES_PDB
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
$ORACLE_UNQNAME=O19CDR
```

---

### VERIFY PDB STATUS ON STANDBY DB - PART 1

```
# on : odr19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR4:SYS@SQL>


-- change container to ROOT CDB
ALTER SESSION SET container=CDB$ROOT ;


COLUMN pdb_name FORMAT a30
COLUMN recovery FORMAT a15

SELECT name pdb_name, recovery_status recovery FROM v$pdbs ;


-- output :
------------------------------ ---------------
PDB_NAME                       RECOVERY
------------------------------ ---------------
PDB$SEED                       ENABLED
SALES_PDB                      DISABLED
------------------------------ ---------------

-- as you can see here, SALES_PDB has RECOVERY=DISABLED.
-- this means, our STANDBY DB is not applying any REDO LOGs from PRIMARY DB for this PDB.

```


### VERIFY PDB STATUS ON STANDBY DB - PART 2

```
# on : odr19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR4:SYS@SQL>


-- change container to SALES_PDB
ALTER SESSION SET container=SALES_PDB ;


COLUMN datafile_name FORMAT a99
COLUMN status        FORMAT a15

-- SELECT file#, name datafile_name, status FROM v$datafile ORDER BY 1 ;
SELECT status, count(1) no_of_datafiles FROM v$datafile GROUP BY status ORDER BY 1 ;


-- output :
--------------- ---------------
STATUS          NO_OF_DATAFILES
--------------- ---------------
RECOVER                      10
SYSOFF                        1
--------------- ---------------

-- STATUS=RECOVER : means these datafiles will require recovery. After we finish configuring RECOVERY via STANDBY process ... this STATUS will become ONLINE.
-- STATUS=SYSOFF  : means this is a datafile for SYSTEM tablespace and it requires recovery. After we finish configuring RECOVERY via STANDBY process ... this STATUS will become SYSTEM.

```

---

### SET ARCHIVELOG DELETION POLICY TO NONE

##### on both PRIMARY DB and STANDBY DB
##### using RMAN
```
CONFIGURE ARCHIVELOG DELETION POLICY TO NONE ;
```
##### this is to make sure, we will not have MISSING ARCHIVELOG FILES during standby build for our new PLUGGABLE DATABASE (PDB).

---

### SOURCE PDB OPEN IN READ WRITE MODE

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
WHERE name = 'SALES_PDB'
ORDER BY con_id, inst_id ;


-- make sure OPEN_MODE=READ WRITE
-- if it is not, then open it in READ WRITE mode

-- close PDB
alter pluggable database SALES_PDB close instances=ALL ;

-- open PDB for READ WRITE
alter pluggable database SALES_PDB open instances=ALL ;

```

---

### RMAN RESTORE PDB ON STANDBY DB

```
# on : odr19d.OracleByExample.com


$ vi /tmp/SALES_PDB_duppdb_from_primary.rmn
#########
run {
  # rman will allow upto 64 channels
  # higher number of channels will give higher performance and parallelism and will require higher amount of resource too
  ALLOCATE CHANNEL disk_01 DEVICE TYPE disk ;
  ALLOCATE CHANNEL disk_02 DEVICE TYPE disk ;
  ALLOCATE CHANNEL disk_03 DEVICE TYPE disk ;
  ALLOCATE CHANNEL disk_04 DEVICE TYPE disk ;

  SET NEWNAME FOR PLUGGABLE DATABASE sales_pdb TO new ;
  RESTORE PLUGGABLE DATABASE sales_pdb FROM SERVICE O19CPR SECTION SIZE 8G ;
}
#########



$ vi /tmp/SALES_PDB_duppdb_from_primary.sh
#########
export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"
export RMAN_LOGFILE=/tmp/SALES_PDB_duppdb_from_primary_`date +'%Y%m%d-%H%M'`.log
rm $RMAN_LOGFILE

date +'%Y-%m-%d %H:%M' >> $RMAN_LOGFILE
$ORACLE_HOME/bin/rman target / cmdfile /tmp/SALES_PDB_duppdb_from_primary.rmn >> $RMAN_LOGFILE
date +'%Y-%m-%d %H:%M' >> $RMAN_LOGFILE
#########



# execute the shell script
$ sh /tmp/SALES_PDB_duppdb_from_primary.sh

```


### STOP REDO APPLY ON STANDBY DB

```
# on : odr19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR4:SYS@SQL>


-- change container to ROOT CDB
ALTER SESSION SET container=CDB$ROOT ;

-- check for recovery process
SELECT inst_id, process, thread#, sequence#, status FROM gv$managed_standby WHERE process LIKE 'MRP%' ;

-- stop REDO APPLY if it is already running
ALTER DATABASE recover managed standby database CANCEL ;

```


### RMAN SWITCH PDB TO COPY

```
# on : odr19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4


$ORACLE_HOME/bin/rman target /

RMAN>
SWITCH PLUGGABLE DATABASE sales_pdb TO copy ;


### sample output :
#########
using target database control file instead of recovery catalog
datafile 561 switched to datafile copy "+ASM_FOR_DATA/SALES_PDB/EC3C057ED5430A40E0537B5F193F87CC/DATAFILE/system.25254.1101742787"
datafile 562 switched to datafile copy "+ASM_FOR_DATA/SALES_PDB/EC3C057ED5430A40E0537B5F193F87CC/DATAFILE/sysaux.25255.1101742789"
datafile 563 switched to datafile copy "+ASM_FOR_DATA/SALES_PDB/EC3C057ED5430A40E0537B5F193F87CC/DATAFILE/sysaux.25256.1101742791"
datafile 564 switched to datafile copy "+ASM_FOR_DATA/SALES_PDB/EC3C057ED5430A40E0537B5F193F87CC/DATAFILE/undotbs1.25257.1101742793"
#########

```


### GENERATE DATAFILES ONLINE SCRIPT

```
# on : odr19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR4:SYS@SQL>


-- change container to SALES_PDB
ALTER SESSION SET container=SALES_PDB ;


-- generate script
set linesize 200
set pagesize 9999
set echo off
set timing off
set heading off
set feedback off

spool /tmp/SALES_PDB_online_files.sql

SELECT 'ALTER DATABASE DATAFILE ' || lpad(to_char(file#), 15, ' ') || ' ONLINE ;' FROM v$datafile ;

spool off

exit ;

```


### MAKE PDB DATAFILES ONLINE

```
# on : odr19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4


-- stop CDB on all nodes
$ srvctl stop database -db O19CDR


$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR4:SYS@SQL>

-- startup database in MOUNT mode
STARTUP mount ;

-- change container to SALES_PDB
ALTER SESSION set container=SALES_PDB ;

-- enable recovery for SALES_PDB
ALTER PLUGGABLE DATABASE enable recovery ;

-- execute this script which was generated in previous step
-- this will make all PDB related datafiles ONLINE
@/tmp/SALES_PDB_online_files.sql

-- change container to ROOT CDB
ALTER SESSION SET container=CDB$ROOT ;

-- shutdown the database
SHUTDOWN immediate ;

exit ;

```


### START STANDBY DB AND START REDO APPLY

```
# on : odr19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4


-- start CDB on all nodes
$ srvctl start database -db O19CDR

-- check status of CDB
$ srvctl status database -db O19CDR


$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR4:SYS@SQL>

-- change container to ROOT CDB
ALTER SESSION SET container=CDB$ROOT ;

-- start REDO APPLY process
ALTER DATABASE recover managed standby database disconnect from session ;

-- check for recovery process
SELECT inst_id, process, thread#, sequence#, status FROM gv$managed_standby WHERE process LIKE 'MRP%' ;

```


### OPEN PDB FOR READ ONLY ON STANDBY DB

#### NOTE : ORACLE ACTIVE DATA GUARD WILL REQUIRE SPECIAL LICENSES

```
# on : odr19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4


$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR4:SYS@SQL>

-- change container to ROOT CDB
ALTER SESSION SET container=CDB$ROOT ;

-- open PDB for READ ONLY
ALTER PLUGGABLE DATABASE sales_pdb OPEN READ ONLY INSTANCES=all ;


-- change container to SALES_PDB
ALTER SESSION set container=SALES_PDB ;

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

---

### VERIFY PDB STATUS ON STANDBY DB - PART 1

```
# on : odr19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR4:SYS@SQL>


-- change container to ROOT CDB
ALTER SESSION SET container=CDB$ROOT ;


COLUMN pdb_name FORMAT a30
COLUMN recovery FORMAT a15

SELECT name pdb_name, recovery_status recovery FROM v$pdbs ;


-- output :
------------------------------ ---------------
PDB_NAME                       RECOVERY
------------------------------ ---------------
PDB$SEED                       ENABLED
SALES_PDB                      ENABLED
------------------------------ ---------------

-- as you can see here, SALES_PDB now has RECOVERY=ENABLED.
-- this means, our STANDBY DB will apply REDO LOGs from PRIMARY DB for this PDB.

```


### VERIFY PDB STATUS ON STANDBY DB - PART 2

```
# on : odr19d.OracleByExample.com

export ORACLE_BASE=/mnt01/oracle/
export ORACLE_HOME=/mnt01/oracle/product/DBHome1911
export ORACLE_UNQNAME=O19CDR
export ORACLE_SID=O19CDR4

$ORACLE_HOME/bin/sqlplus / as sysdba
O19CDR4:SYS@SQL>


-- change container to SALES_PDB
ALTER SESSION SET container=SALES_PDB ;


COLUMN datafile_name FORMAT a99
COLUMN status        FORMAT a15

-- SELECT file#, name datafile_name, status FROM v$datafile ORDER BY 1 ;
SELECT status, count(1) no_of_datafiles FROM v$datafile GROUP BY status ORDER BY 1 ;


-- output :
--------------- ---------------
STATUS          NO_OF_DATAFILES
--------------- ---------------
ONLINE                       10
SYSTEM                        1
--------------- ---------------

```

---

### SET ARCHIVELOG DELETION POLICY

##### on both PRIMARY DB and STANDBY DB
##### using RMAN
```
CONFIGURE ARCHIVELOG DELETION POLICY TO <whatever_you_need_for_your_environment> ;
```

---
