# The ultimate golden release upgrade guide, 9.2 edition

How to get from dCache 8.2 to dCache 9.2

## Executive summary

### The highlights in 9.2 compared with 8.2

- dCache now supports Java 17 as its platform
- Improve concurrent file create rate in a single directory
- Performance improvements for the concurrent directory creation and removal


## Breaking changes

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



