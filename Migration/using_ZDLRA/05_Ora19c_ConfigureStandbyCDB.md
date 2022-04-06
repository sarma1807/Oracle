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

```
Creating a Physical Standby using RMAN Duplicate (RAC or Non-RAC) (Oracle Support Doc ID 1617946.1)
Step by Step Guide on Creating Physical Standby Using RMAN DUPLICATE...FROM ACTIVE DATABASE (Oracle Support Doc ID 1075908.1)
```

### DISABLE Data Guard Broker
##### we will NOT configure/use Data Guard Broker for this cluster

```
# on : ora19a.OracleByExample.com

O19CDB1:SYS@SQL>

SHOW PARAMETER dg

NAME                    TYPE     VALUE
----------------------- -------- ------------------------------
dg_broker_config_file1  string   /mnt01/oracle/product/DBHome1911/dbs/dr1O19CDB.dat
dg_broker_config_file2  string   /mnt01/oracle/product/DBHome1911/dbs/dr2O19CDB.dat
dg_broker_start         boolean  FALSE


ALTER SYSTEM SET dg_broker_config_file1='+ASM_FOR_DATA/O19CDB/dr1O19CDB.dat' SCOPE=both SID='*' ;
ALTER SYSTEM SET dg_broker_config_file2='+ASM_FOR_DATA/O19CDB/dr2O19CDB.dat' SCOPE=both SID='*' ;
ALTER SYSTEM SET dg_broker_start=FALSE                                       SCOPE=both SID='*' ;
```
