------------------------------------------------------------------------

-- ORACLE BASE and HOME based on 19c docs

-- for ORACLE INVENTORY
ORACLE_INVENTORY=/u01/app/oraInventory

-- for GI
ORACLE_BASE=/u01/app/grid
ORACLE_HOME=/u01/app/19.0.0/gridHome_01

-- GI_BASE will store ASM and Clusterware/GI related diagnostic, administrative and other logs.
-- GI_HOME must not be placed under GI_BASE. GI_HOME will be owned by root user.

-- for RDBMS
ORACLE_BASE=/u01/app/oracle
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbHome_01

-- ORACLE_BASE will store RDBMS related diagnostic, administrative and other logs, including scripts related to databases.
-- By default, ORADATA and FAST_RECOVERY_AREA will also be stored here incase of non-ASM based databases.
-- according to OFA : ORACLE_HOME must be placed under ORACLE_BASE.
-- GI_HOME and ORACLE_HOME must be in different paths to avoid any directory ownership conflicts.

------------------------------------------------------------------------

-- for RAC

mkdir -p /u01/app/19.0.0/gridHome_01
mkdir -p /u01/app/grid
mkdir -p /u01/app/oracle
mkdir -p /u01/app/oracle/product/19.0.0/dbHome_01
mkdir -p /u01/app/oraInventory

chown -R grid:oinstall /u01
chown -R oracle:oinstall /u01/app/oracle
chown -R grid:oinstall /u01/app/oraInventory

chmod -R 775 /u01

------------------------------------------------------------------------

-- for single-instance
chown -R oracle:oinstall /u01
chown -R oracle:oinstall /u01/app/oracle
chown -R oracle:oinstall /u01/app/oraInventory

mkdir -p /u02/app/oracle/oradata
mkdir -p /u03/app/oracle/fra

chown -R oracle:oinstall /u02/app/oracle
chown -R oracle:oinstall /u03/app/oracle

chmod -R 775 /u02
chmod -R 775 /u03

------------------------------------------------------------------------
