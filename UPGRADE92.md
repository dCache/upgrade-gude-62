# The ultimate golden release upgrade guide

### Community effort to write an upgrade guide for dCache 9.2

How to get from dCache 8.2 to dCache 9.2

## Executive summary

### The highlights in 9.2 compared with 8.2

- dCache now supports Java 17 as its platform
- Improve concurrent file create rate in a single directory
- Performance improvements for the concurrent directory creation and removal


## Breaking changes

### WARNING -- Incompatibility between 8.2 and 9.2

Consequences:
- When upgrading from 8.2 to 9.2, you need to upgrade the entire instance
- If you run srm-manager, you need to clean `/var/lib/dcache/credentials`, `srmrequestcredentials` and  all entries in the `*requests` and `*filerequests` tables from the srm
database  
  

### General incompatibilities

- DCAP and NFS doors will fail the request if file’s storage unit is not configured in PoolManager
- linklocal and localhost interfaces are not published by doors and pools
- DCAP movers always start in passive mode
- removed experimental message encoding format
- removed default HSM operation timeout
- Starting version 9.1 the nlink count for directories shows only number of subdirectories. Thus, the existing nlink count can be out-of-sync with we no automatic re-synchronization is performed.
- dropped gplazma support for XACML
- pool binds TCP port for http and xroot movers on startup
- The `cleaner` cell name no longer exists, the service now consists of two cells: `cleaner-disk` and `cleaner-hsm`

### Bulk

-   The storage layer has been redesigned to conserve space and for efficiency/throughput.
    Moving up to the new database schema may require some time.  The amount of time can be 
    generally computed in terms of the number of entries in the `request_target` table; 
    figure about 1 hour for every 10 million.  If it is necessary to maintain all such entries, 
    we recommend you do the upgrade offline, using the `dcache database update` command-line tool.  
    If there is no need to keep completed requests in the database, we would advise truncation 
    or at least deletion of the completed requests before the upgrade.

-   The way that the containers are managed has also beem significantly modified.   The visible
    changes have to do with properties.   First, the `max-permits` properties on the individual
    activities are no longer used.  Throttling of activity operations is achieved using 
    the `bulk.limits.dir-list-semaphore` and `bulk.limits.in-flight-semaphore`
    values, along with rate limiters on the endpoints (`bulk.limits.pin-manager-rate-per-second`,
    `bulk.limits.pnfs-manager-rate-per-second`, `bulk.limits.qos-engine-rate-per-second`).  
    The thread pools have also changed somewhat; `bulk.limits.delay-clear-threads`, 
    `bulk.limits.dir-list-threads` and `bulk.limits.activity-callback-threads` are no longer used.   
    
    It is anticipated that adjustments to these defaults should not be necessary under normal loads.

-   Periodic archiving of requests has been added; this is configurable via properties
    and admin commands.  The properties:
    
    `bulk.limits.archiver-window`
    `bulk.limits.archiver-window.unit`
    `bulk.limits.archiver-period`
    `bulk.limits.archiver-period.unit`

    determine the arrival time (from now) in the past before which terminated requests 
    will be cleared when the archiver runs, and the frequency with which it runs, respectively.
    To reset these, use

    `\s bulk archiver reset`
    
    The archive table maintains an abbreviated summary of the 
    requests, and can be purged via admin command as well.

    `\s request archived ls`
    `\s request archived clear`

    Whenever you run `\s request clear`, a summary entry is also written to the archive table 
    before deletion from the main tables.

    Depending on the volume of activity, it may be wise to tighten the period and window
    for archiving (the defaults are very conservative).   As stated in the release notes,
    the `clearOnSuccess` and `clearOnFailure` options for individual requests are not
    available from `STAGE` requests (the TAPE API), so the archiver is the only way
    to clean these up automatically.

-   Activity providers now can capture the environment so that defaults can be 
    customized (this presently pertains only to PIN and STAGE lifetime attributes).
    
    See:

    `bulk.plugin!pin.default-lifetime`
    `bulk.plugin!pin.default-lifetime.unit`
    `bulk.plugin!stage.default-lifetime`
    `bulk.plugin!stage.default-lifetime.unit`

-   Support has been added for the QoS update request to handle policy arguments. Note that `targetQos` is
    still valid.  Please see the new cookbook section on QoS policies for details.

    ```
    \s bulk arguments UPDATE_QOS

    NAME        |   DEFAULT |             VALUE SPEC |  DESCRIPTION
    qosPolicy   |      null |   string, max 64 chars |  the name of the qos policy to apply to the file
    qosState    |         0 |                integer |  the index into the desired policy's list states
    targetQos   |      null |    disk|tape|disk+tape |  the desired qos transition ('disk' is limited to files with volatile/unknown qos status)

    ```

-   Bulk has been made replicable (HA).  Please read the requirements
    in the cookbook section on High Availability.   The following property controls the timeout for a
    response from the leader:

    `bulk.service.ha-leader.timeout`
    `bulk.service.ha-leader.timeout.unit`

-   The `delay clear` option has been eliminated from bulk requests.

-   The `prestore` option has also been eliminated, as it is no longer necessary
    (since all initial targets are immediately batch stored synchronously)
    before the submission request returns.

-   NOTE: Running more concurrent containers in Bulk means their files will be sliced. 
          If it is important to keep most files in a request together for the purposes of staging optimization,
          then the max concurrency in Bulk should be turned down. It could even be set at 1 
	  (essentially one request at a time) if necessary.

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

### Frontend

-   Support for .well-known/security.txt was added to both the frontend and WebDav ports.  The dcache properties
    `dcache.wellknown!wlcg-tape-rest-api.path` and `dcache.wellknown!security-txt.uri` apply globally; 
    `frontend.wellknown!wlcg-tape-rest-api.path` has been deprecated.
-   Support for relative paths and symlink prefix resolution for bulk and namespace resources.
-   Authz checks have been removed in Quota GET methods.
-   Use of RolePrincipal (see under gPlazma) replaces reliance on LoginAttributes and the
    old admin role (special gid) as defined by the roles plugin.
-   Support for QoS Rule Engine policies.  A new qos-policy resource allows one to
    add, remove, list and retrieve policy definitions.  See the Swagger pages for details.
-   An -optional query parameter was added to the namespace resource to retrieve extra information 
    about a file; the new QoS Policy file attributes (QOS_POLICY and QOS_STATE) are included
    with this option.
-   Support for pool migration has been introduced (migrations resource).
    See Swagger pages for details.

### GPlazma

With the addition of the RolePrincipal, the `roles` plugin may be considered deprecated.  In 9.2, neither 
Frontend nor dCacheView make use of it any more.   Instead, one must grant privileges by using the multimap plugin.

An example:

```
dn:"/DC=org/DC=cilogon/C=US/O=Fermi National Accelerator Laboratory/OU=People/CN=Al Rossi/CN=UID:arossi" username:arossi  uid:8773  gid:1530,true roles:admin
```

Currently there are three roles defined: admin, qos-user and qos-group.  The second allows
the user to transition files owned by the user's uid; the third allows the user
to transition files whose group is the user's primary gid.  The two qos roles
can be combined.  Admin grants full admin privileges.

### PnfsManager

To remove unused directory tags chimera keeps the reference count (nlink) of tags. This approach creates a ‘hot’ record that serializes all updates to a given top-level tag. Starting 9.2 dCache doesn’t
rely on ref count anymore and uses conditional DELETE, which should improve the concurrent directory creation/deletion rate.

### NFS

Prior to version 9.2 dCache, to support RHEL6-based clients, if no export options are specified, the NFSv4.1 door was publishing only the nfs4_1_files layout. Now on the door publishes all available layout types. If for whatever reason RHEL6 clients are still used, the old behaviour can be enforced by the `lt=nfsv4_1_files` export option.

### Pool

Typically, when dCache interacts with an HSM, there is a timeout on how long such requests can stay in the HSM queue. Despite the fact that those timeouts are HSM-specific, dCache comes with its own default values, which are usually incorrect, so admins usually end up explicitly setting them. Starting with version 9.0, the default timeout has been removed. This means that there is no timeout for HSM operations unless explicitly set by admins.

>    NOTE: this change is unlikely to break existing setups, as previous timeout values are already stored in the pool setup file.

In addition, a new command `sh|rh|rm unset timeout` has been added to drop defined timeouts.

### QoS

-   `QoS was made to support migration using a new pool mode, "DRAINING"; please consult the
    book chapter for further details.

-   The DB namespace endpoint can now be configured to be separate from the main Chimera
    database (originally introduced/changed for Resilience). In this way, the scanner,
    whose namespace queries are read-only, could be pointed at a database replica.

    This remains possible for QoS, even though the QoSEngine now also is responsible
    for updating Chimera with file policy state.  These writes are all done via
    messaging (PnfsHandler) rather than by direct DB connection, so they go to
    the master Chimera instance.

-   Requests for QoS transitions are now authorized on the basis of role (see under gPlazma).

-   The first version of the QoS Rule Engine has been added.   With this, one can
    define a QoS policy to apply to files either through a directory tag or via
    a requested transition; the engine tracks the necessary changes in state over time.
    A new database table has been added to the qos database.  Remember that if you are
    deploying QoS for the first time, you need to create the database:

    createdb -U \<user\> qos
    
    Please consult the cookbook chapter on QoS policies for further details.

-   In conformity with the new rule engine changes, the scanner has been modified in terms of
    how it runs scans.
    
    The scan period refers to the default amount of time between sweeps to check for timeouts.
    
    The scan windows refer to the amount of time between scheduled periodic system diagnostic scans.

    QOS NEARLINE refers to files whose QoS policy is defined and whose RP is NEARLINE CUSTODIAL.
    ONLINE refers to scans of all files with persistent copies, whether or not they are 
    REPLICA or CUSTODIAL.

    ONLINE scanning is done by a direct query to the namespace, and is batched
    into requests determined by the batch size.  Unlike with resilience, this
    kind of scan will only touch each inode entry once (whereas pool scans may overlap
    when multiple replicas are involved).
    
    On the other hand, a general pool scan will only look at files on pools that are
    currently IDLE and UP, so those that are excluded or (temporarily) unattached
    will be skipped.   This avoids generating a lot of alarms concerning files without
    disk copies that should exist.
    
    The direct ONLINE scan is enabled by default.  To use the pool scan instead, disable
    "online" either via the property or the admin reset command.  Be aware, however, that
    unlike resilience, all pools will be scanned, not just those in the resilient/primary
    groups; thus the online window should be set to accommodate the amount of time it
    will take to cycle through the entire set of pools this way. Needless to say, doing
    a direct ONLINE scan probably will take less time than a general pool scan.

    The batch size for a direct ONLINE scan is lowered to serve as an implicit backgrounding or
    de-prioritization (since the scan is done in batches, this allows for preemption by
    QOS scans if they are running concurrently).

    The relevant properties;

    ```
    qos.limits.scanner.scan-period
    qos.limits.scanner.scan-period.unit
    qos.limits.scanner.qos-nearline-window
    qos.limits.scanner.qos-nearline-window.unit
    qos.limits.scanner.enable.online-scan
    qos.limits.scanner.online-window=2
    qos.limits.scanner.online-window.unit
    qos.limits.scanner.qos-nearline-batch-size
    qos.limits.scanner.online-batch-size
    
    ```

    More details available in the Book.  Scans can also be triggered manually:

    ```
    \h sys scan 

    NAME
       sys scan -- initiate an ad hoc background scan.

    SYNOPSIS
       sys scan [-online] [-qos] 

    DESCRIPTION
       If a scan of the requested type is already running, it will not
       be automatically canceled.

    OPTIONS
         -online
              Scan online files (both REPLICA and CUSTODIAL). Depending
              on whether online is enabled (true by default), it will
              either scan the namespace entries or will trigger a scan
              of all IDLE ENABLED pools. Setting this scan to be run
              periodically should take into account the size of the
              namespace or the number of pools, and the proportion of
              ONLINE files they contain. 
         -qos
              Scan NEARLINE files for which a QoS policy has been
              defined.
    ```

Note that the singleton QoS service (where all four components are plugged into each other 
directly) is no longer available; the four services can, however, still be run together or in
separate domains, as with any dCache cell.

### Resilience

Resilience is still available in 9.2, but should be considered as superseded by the
QoS services.   We encourage you to switch to the latter as soon as is feasible.
Remember *_not_* to run Resilience and QoS simultaneously.

### XRootD

- Proxying through the xroot door is now available.  See the following properties:

```
#
#   Proxy all transfers through the door.
#
xrootd.net.proxy-transfers=false

#   What IP address to use for connections from the xroot door to pools.
#
#   When the data transfer is proxied through the door, this property is
#   used as a hint for the pool to select an interface to listen to for
#   the internal connection created from the door to the pool.
#
xrootd.net.internal=

#
#   Port range for proxied connections.
#
xrootd.net.proxy.port.min=${dcache.net.wan.port.min}
xrootd.net.proxy.port.max=${dcache.net.wan.port.max}

#
#   How long to wait for a response from the pool.
#
xrootd.net.proxy.response-timeout-in-secs=30
```

- Relative paths are supported in the xroot URL.   Resolution of paths will be done
  on the basis of the user root as defined in the gPlazma configuration files.  

  We encourage you not to use `xrootd.root` if possible.

- Resolution of symlinks in path prefixes and paths is supported.

- The efficiency of the stat list (ls -l) has been greatly improved.

## Runtime environment

## New services

