# libra.so

### To interact with ZDLRA, first thing we need is the `libra.so` file

```
ZDLRA: Where to download new sbt library (libra.so module) (Oracle Support Doc ID 2219812.1)
```
```
RA = Recovery Appliance
SBT = Serial Backup Tape
.so = Linux compiled library file ("Shared Object")
```

We need to download this very small 91mb `libra.so` file and place it in `$ORACLE_HOME/lib/` folder
```
$ ls -lh $ORACLE_HOME/lib/libra.so
-rw-r--r-- 1 oracle oinstall 91M Mar 21 2022 /mnt01/oracle/product/DBHome11204/lib/libra.so
$
```

`sbttest` to verify `libra.so`
```
$ $ORACLE_HOME/bin/sbttest version -libname $ORACLE_HOME/lib/libra.so
The sbt function pointers are loaded from /mnt01/oracle/product/DBHome11204/lib/libra.so library.
-- sbtinit succeeded
-- sbtinit (2nd time) succeeded
sbtinit: vendor description string=Oracle Secure Backup
sbtinit: Media manager is version 19.0.0.1
sbtinit: Media manager supports SBT API version 2.0
sbtinit: allocated sbt context area of 1120 bytes
RA Library
Copyright (c) 2008, 2017, Oracle and/or its affiliates.  All rights reserved.
Release :    19.0.0.0.0 Production.
Build label: RDBMS_19.3.0.0.0DBBKPCSBP_LINUX.X64_200922
$
```
```
VERSION >>>>>>>> sbtinit: Media manager is version 19.0.0.1
  BUILD >>>>>>>> Build label: RDBMS_19.3.0.0.0DBBKPCSBP_LINUX.X64_200922
```

---

# Wallet to Connect to ZDLRA

### We will connect to ZDLRA using credentials stored in a wallet

```
# create empty wallet

export WALLET_LOCATION=$ORACLE_HOME/dbs/wallet

mkdir $WALLET_LOCATION

$ORACLE_HOME/bin/orapki wallet create -wallet $WALLET_LOCATION -auto_login_only
Oracle PKI Tool : Version 11.2.0.4.0 - Production
Copyright (c) 2004, 2013, Oracle and/or its affiliates. All rights reserved.
$

ls -lh $WALLET_LOCATION
-rw------- 1 oracle oinstall 3.6K Mar 21 2022 cwallet.sso
$
```

```
# add zdlra credentials to wallet

export WALLET_FILE=$WALLET_LOCATION/cwallet.sso


$ORACLE_HOME/bin/mkstore -wrl $WALLET_FILE -createCredential CONNECT_TO_ZDLRA rasys_user strong_password
Oracle Secret Store Tool : Version 11.2.0.4.0 - Production
Copyright (c) 2004, 2013, Oracle and/or its affiliates. All rights reserved.

Create credential oracle.security.client.connect_string1
$


# list credentials stored in wallet

$ORACLE_HOME/bin/mkstore -wrl $WALLET_FILE -listCredential
Oracle Secret Store Tool : Version 11.2.0.4.0 - Production
Copyright (c) 2004, 2013, Oracle and/or its affiliates. All rights reserved.

List credential (index: connect_string username)
1: CONNECT_TO_ZDLRA rasys_user
$
```
