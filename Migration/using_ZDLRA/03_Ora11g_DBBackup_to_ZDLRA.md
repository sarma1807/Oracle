# On Source 11.2.0.4 Database

### Block Change Tracking
##### improves the performance of incremental backups by recording changed blocks in the block change tracking file. During an incremental backup, instead of scanning all data blocks to identify which blocks have changed, RMAN uses this file to identify the changed blocks that need to be backed up.

### check if "Block Change Tracking" is enabled

```
SQL>
SELECT status, ceil(bytes/(1024*1024)) size_mb, filename FROM v$block_change_tracking ;
```
##### output if "Block Change Tracking" is DISABLED
```
STATUS          SIZE_MB FILENAME
---------- ------------ --------------------------------------------------
DISABLED
```

### ENABLE "Block Change Tracking"
##### enabling "Block Change Tracking" is optional. if enabled, it will speed up the incremental backup process.

```
-- if DB_CREATE_FILE_DEST is already set while using ASM or in RAC (BCT file can be written to ASM Disk Group)
SQL>
ALTER DATABASE ENABLE block change tracking ;
```

```
-- if DB_CREATE_FILE_DEST is not set and using ASM
SQL>
ALTER DATABASE ENABLE block change tracking USING FILE '+ASM_DATA' ;
```

```
-- if DB_CREATE_FILE_DEST is not set and NOT using ASM
-- REUSE clause, if file already exists
SQL>
ALTER DATABASE ENABLE block change tracking USING FILE '/oradata/SALES_DB/block_change_tracking.file' [REUSE] ;
```

### check if "Block Change Tracking" is enabled

```
SQL>
SELECT status, ceil(bytes/(1024*1024)) size_mb, filename FROM v$block_change_tracking ;
```
##### output if "Block Change Tracking" is ENABLED
```
STATUS          SIZE_MB FILENAME
---------- ------------ --------------------------------------------------
ENABLED             321 +ASM_DATA/SALES/changetracking/ctf.321.2022032100
```


### DISABLE "Block Change Tracking"
##### if you ever want to disable "Block Change Tracking"

```
SQL>
ALTER DATABASE DISABLE block change tracking ;
```

---
### now it is time to work with our Oracle ZDLRA administrators to get this database registered with ZDLRA
##### due to complexity and product licensing policies, this process is NOT being documented here.
---

### RMAN configuration changes

```
$ export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"

### rman catalog connects to ZDLRA using Wallet, which we have configured in the previous step
$ rman target / catalog /@CONNECT_TO_ZDLRA
RMAN>

CONFIGURE CHANNEL DEVICE TYPE 'SBT_TAPE' FORMAT '%d_%U' PARMS "SBT_LIBRARY=/mnt01/oracle/product/DBHome11204/lib/libra.so, SBT_PARMS=(RA_WALLET='location=file:/mnt01/oracle/product/DBHome11204/dbs/wallet credential_alias=CONNECT_TO_ZDLRA')" ;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE SBT_TAPE TO '%F' ;
CONFIGURE DEVICE TYPE 'SBT_TAPE' PARALLELISM 8 BACKUP TYPE TO BACKUPSET ;
CONFIGURE DATAFILE BACKUP COPIES FOR DEVICE TYPE SBT_TAPE TO 1 ;
CONFIGURE ARCHIVELOG BACKUP COPIES FOR DEVICE TYPE SBT_TAPE TO 1 ;

resync catalog ;
```

### DB Incremental Level ZERO Backup
##### we need to perform this backup only once

```
$ export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"
$ rman target / catalog /@CONNECT_TO_ZDLRA
RMAN>

backup incremental level 0 cumulative 
device type sbt 
filesperset = 3 
tag 'zdlra_incremental_level_zero_backup' 
section size 32G 
database ;
```

##### first backup will take some time, but future incremental level 1 backups will run fast

---
