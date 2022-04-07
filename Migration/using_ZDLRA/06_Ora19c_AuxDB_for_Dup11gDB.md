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
# on ora19d.OracleByExample.com

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
# on ora19d.OracleByExample.com

$ORACLE_HOME/bin/orapwd FILE='/mnt01/oracle/product/DBHome1911/dbs/orapwAUXDB' FORMAT=12.2
Enter password for SYS: <PASSWORD_REMOVED>

```
