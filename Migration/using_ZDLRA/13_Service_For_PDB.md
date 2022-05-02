## srvctl add service for PDB
https://docs.oracle.com/en/database/oracle/oracle-database/19/racad/server-control-utility-reference.html#GUID-EC1BA6D7-D538-4E11-9B31-C59389FDF93B

---

#### as GRID user :

```
# create service
srvctl add service -db <CONTAINER_DB_NAME> -service <"SERVICE_NAME_FOR_PDB"> -preferred <"RAC_NODES_COMMA_SEPARATED"> -failovertype AUTO -pdb <PDB_NAME> -notification TRUE -stopoption IMMEDIATE


# verify service configuration
srvctl config service -db <CONTAINER_DB_NAME>
srvctl config service -db <CONTAINER_DB_NAME> -service <SERVICE_NAME_FOR_PDB>


# enable/disable service
srvctl enable  service -db <CONTAINER_DB_NAME> -service <SERVICE_NAME_FOR_PDB>
srvctl disable service -db <CONTAINER_DB_NAME> -service <SERVICE_NAME_FOR_PDB>


# start/stop service
srvctl start service -db <CONTAINER_DB_NAME> -service <SERVICE_NAME_FOR_PDB>
srvctl stop  service -db <CONTAINER_DB_NAME> -service <SERVICE_NAME_FOR_PDB>


# remove service
srvctl remove  service -db <CONTAINER_DB_NAME> -service <SERVICE_NAME_FOR_PDB>

```

---

