# on New PRIMARY Cluster

```
# 4 Node RAC built using Oracle 19.11.0.0.0 Grid Infrastructure and RDBMS

ora19a.OracleByExample.com
ora19b.OracleByExample.com
ora19c.OracleByExample.com
ora19d.OracleByExample.com

$ORACLE_BASE=/mnt01/oracle/
$ORACLE_HOME=/mnt01/oracle/product/DBHome1911
```

### Create a new Container DB (CDB) using DBCA

```
Global DB Name : O19CPR
SID Prefix     : O19CPR
Use Local UNDO for Pluggable Databases (PDBs)
Create as Container DB without any PDBs (Empty Container DB)
```
