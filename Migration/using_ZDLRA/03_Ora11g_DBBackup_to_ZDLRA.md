# On Source 11.2.0.4 Database

---

### Block Change Tracking : improves the performance of incremental backups by recording changed blocks in the block change tracking file. During an incremental backup, instead of scanning all data blocks to identify which blocks have changed, RMAN uses this file to identify the changed blocks that need to be backed up.

### check if Block Change Tracking is enabled

```
SQL>
SELECT status, ceil(bytes/(1024*1024)) size_mb, filename FROM v$block_change_tracking ;
```
##### output if Block Change Tracking is DISABLED
```
STATUS          SIZE_MB FILENAME
---------- ------------ --------------------------------------------------
DISABLED
```
