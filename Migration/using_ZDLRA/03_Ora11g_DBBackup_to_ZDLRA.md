# On Source 11.2.0.4 Database

---

##### Block Change Tracking : improves the performance of incremental backups by recording changed blocks in the block change tracking file. During an incremental backup, instead of scanning all data blocks to identify which blocks have changed, RMAN uses this file to identify the changed blocks that need to be backed up.

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
