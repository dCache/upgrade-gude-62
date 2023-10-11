# The ultimate golden release upgrade guide, 9.2 edition

How to get from dCache 8.2 to dCache 9.2

## Executive summary

### The highlights in 9.2 compared with 8.2

- dCache now supports Java 17 as its platform
- Improve concurrent file create rate in a single directory
- Performance improvements for the concurrent directory creation and removal


## Breaking changes

### General incompatibilities

- the cleaner service, originally a single cell, now consists of two parts: one cell for disk cleaning (cleaner-disk), one for hsm cleaning (cleaner-hsm)
- DCAP and NFS doors will fail the request if file’s storage unit is not configured in PoolManager
- linklocal and localhost interfaces are not published by doors and pools
- DCAP movers always start in passive mode
- removed experimental message encoding format
- removed default HSM operation timeout
- Starting version 9.1 the nlink count for directories shows only number of subdirectories. Thus, the existing nlink count can be out-of-sync with we no automatic re-synchronization is performed.
- dropped gplazma support for XACML
- pool binds TCP port for http and xroot movers on startup



### Cleaner

The `cleaner` service, originally a single cell, now consists of two parts: one cell for disk cleaning (`cleaner-disk`), one for hsm
cleaning (`cleaner-hsm`). They can be deployed as desired, be assigned different resources and each run in HA mode.
This will hopefully improve performance issues and help admins configure and understand cleaner behaviour.

Be aware that the property names have changed their prefixes from `cleaner.<something>` to `cleaner-disk.<something>` and
`cleaner-hsm.<something>`, while some admin commands have lost the "hsm" String from their name.
Note: As not all previously existing parameters were used to control the behaviour of both the disk and hsm parts of the
old, combined `cleaner` cell, please check which parameters are carried over to `cleaner-hsm` and `cleaner-disk`, respectively.

Example setup:

```
[dCacheDomain]
[dCacheDomain/cleaner-disk]
cleaner-disk.cell.name=cleaner-disk1

[dCacheDomain/cleaner-disk]
cleaner-disk.cell.name=cleaner-disk2

[dCacheDomain/cleaner-hsm]
```

Also, an admin command was added to the hsm-cleaner cell that allows forgetting a tape-resident pnfsid, meaning removing any corresponding delete target entries from the cleaner’s trash table database.



## Runtime environment



## New services



