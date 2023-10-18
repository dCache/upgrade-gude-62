# The ultimate golden release upgrade guide

### Community effort to write an upgrade guide for dCache 9.2

How to get from dCache 8.2 to dCache 9.2

## Executive summary

### The highlights in 9.2 compared with 8.2

- dCache now supports Java 17 as its platform
- Improve concurrent file create rate in a single directory
- Performance improvements for the concurrent directory creation and removal


## Breaking changes

### General incompatibilities

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

### Chimera


Starting dCache version 9.1 chimera allows controlling the behaviour of the parent directory attribute update policy with configuration property `chimera.attr-consistency`, which takes the following values:


| policy | behaviour                                                                                                                                                                                                                                   |
|:-------|----------------------------------------------------------------------------------------------|
| strong | a creation of a filesystem object will right away update the parent directory's mtime, ctime, nlink and generation attributes                                                                                                                   |
| weak   | a creation of a filesystem object will eventually update (after 30 seconds) the parent directory's mtime, ctime, nlink and generation attributes. Multiple concurrent modifications to a directory are aggregated into a single attribute update. |
| soft   | same as weak, however, reading of directory attributes will take into account pending attribute updates.  


Read-write exported NFS doors SHOULD run with strong consistency or soft consistency to maintain POSIX compliance. Read-only NFS doors might run with weak consistency if non-up-to-date directory attributes can be tolerated, for example, when accessing existing data, or soft consistency, if up-to-date information is desired, typically when seeking newly arrived files through other doors.

### PnfsManager

To remove unused directory tags chimera keeps the reference count (nlink) of tags. This approach creates a ‘hot’ record that serializes all updates to a given top-level tag. Starting 9.2 dCache doesn’t
rely on ref count anymore and uses conditional DELETE, which should improve the concurrent directory creation/deletion rate.


### NFS

Prior to version 9.2 dCache, to support RHEL6-based clients, if no export options are specified, the NFSv4.1 door was publishing only the nfs4_1_files layout. Now on the door publishes all available layout types. If for whatever reason RHEL6 clients are still used, the old behaviour can be enforced by the `lt=nfsv4_1_files` export option.

### Pool

Typically, when dCache interacts with an HSM, there is a timeout on how long such requests can stay in the HSM queue. Despite the fact that those timeouts are HSM-specific, dCache comes with its own default values, which are usually incorrect, so admins usually end up explicitly setting them. Starting with version 9.0, the default timeout has been removed. This means that there is no timeout for HSM operations unless explicitly set by admins.

>    NOTE: this change is unlikely to break existing setups, as previous timeout values are already stored in the pool setup file.

In addition, a new command `sh|rh|rm unset timeout` has been added to drop defined timeouts.


## Runtime environment



## New services



