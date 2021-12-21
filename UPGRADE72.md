
# The ultimate golden release upgrade guide

## Community effort to write an upgrade guide for dCache 7.2

How to get from dCache 6.2 to dCache 7.2

## Executive summary

### The highlights in 7.2 compared with 6.2

-  quota system 
-  support for file labels
-  a dedicated scheduling strategy for bring-online requests in SrmManager
-  extended attribute support with http and nfs


## Breaking changes

### Configuration properties
For File system statistics added functionality to cache total files and total space used on DB backend. 
The following two properties has been added:


| property  | new value |
|:----------|-------:|
pnfsmanager.fs-stat-cache.time | 3600
pnfsmanager.fs-stat-cache.time.unit | SECONDS

### New output format in NFS admin interface

The admin interface for NFS doors has been updated to remove redundant information from the output of the `show clients` command:

```
    [dcache-lab000] (NFS-dcache-lab007@core-dcache-lab007) admin > show clients
        /11.19.15.23:967:Linux NFSv4.1 ani:v4.1
            5f4ccad3000300010000000000000001 max slot: 15/0
    
    [dcache-lab000] (NFS-dcache-lab007@core-dcache-lab007) admin >
```

The argument of `kill client` accepts the client's session id. For instance, to kill the client from example above:

```
[dcache-lab000] (NFS-dcache-lab007@core-dcache-lab007) admin > kill client 5f4ccad3000300010000000000000001
```

### Cleaner chunking and parallel cleaning

The HSM cleaner regularly fetches HSM locations for deletion from the trash table and caches them locally for batched dispatch. The maximum number of cached delete locations can now be limited in order to prevent running out of memory if the trash table is too large. The default value is `cleaner.limits.hsm-max-cached-locations = 12000`.

Previously, the disk cleaner could run out of memory if the number of delete locations in the trash table was too large. This is now fixed.

Previously, pools were cleaned synchronously one after another by the cleaner service. Doing so in parallel is expected to provide performance benefits.
The property `cleaner.limits.threads` now also controls the number of pools processed in parallel.

### PinManager unpinning chunking and new pin states

The PinManager service regularly runs background processes for expiring and removing pins. For several reasons, large numbers of pins to be unpinned may accumulate in the PinManager’s trash table, threatening to overwhelm the system due to PinManager trying to execute all unpin tasks in one go. Therefore, chunked unpinning is introduced, with which only a certain number of available unpin tasks will be handled per run. This number can be configured: `pinmanager.max-unpins-per-run=200`

Additionally, two new pin states are added: `READY_TO_UNPIN`, which is the initial state for a pin that should be removed, and `FAILED_TO_UNPIN`, into which a pin transitions when the unpin attempt fails. Only pins in state `READY_TO_UNPIN` are selected for processing.

A new process regularly resets all pins in state `FAILED_TO_UNPIN` back to state `READY_TO_UNPIN` in order to make them eligible to be attempted again. The period in which to reset all pins that failed to unpin can be configured as well:

```
pinmanager.reset-failed-unpins-period=2
pinmanager.reset-failed-unpins-period.unit=HOURS
```

They can be manually reset to state `READY_TO_UNPIN` via the admin command; overall or for a specific pool

```
admin > \s PinManager retry unpinning
```


## Runtime environment

### Java flight recorder

When debugging an issue on a running system often we need to collect jvm performance stats with ‘Java flight recorder’. Starting from release 7.2 the Java flight recorder attach listener is enabled by default. Site admins can collect and provide developers with additional information when high CPU load or memory consuption is observed as:

```
jcmd <pid> JFR.start duration=60s filename=/tmp/dcache.jfr
```

> Please note, that jcmd command is a part of java-11-openjdk-devel package (on RHEL and clones)

### Handling of OutOfMemoryError

Depending which thread have received OutOfMemoryError the JVM might or might not exit. In a later cache, dCache might remain in an unpredictable state, where a component might be exposed as functional when its not.

With 7.2 we have update the java options to include ExitOnOutOfMemoryError, which forces JVM to exit when an OOM is detected.

> NOTE: There are several situations when jvm generates an OutOfMemoryError. The ExitOnOutOfMemoryError option works ONLY when allocation in heap space fails.

### Packaging

Starting from dcache-7.2.5 the rpm package includes `rsyslog` configuration to produce classic log files in `/var/log/dcache` directory in addition to
journal entries. To enable the logging into files the `/etc/systemd/journald.conf` should be configured as:

```
ForwardToSyslog=yes
```

## New services

### SrmManager bring-online scheduling strategy

Optimally recalling data from tape is achieved by reducing the number of tape mounts and on-tape seeks by recalling as much volume as possible per mount. To that end a dedicated scheduling strategy exclusively for bring-online requests is introduced in the SrmManager. It is capable of clustering requests by tape according to a set of configurable criteria before passing them on to the rest of the system. In its current state it requires two files with information on targeted tapes, their capacity and occupancy as well as the mapping of tape-resident files to tape name. The file formats are described in the book.

The scheduler can be activated by adding the following line to the SrmManager section of the properties file:

`srmmanager.plugins.enable-bring-online-clustering = true`

and configured as described in the book.

It is important to note that the scheduler can only be effective when a dCache instance contains exactly one SrmManager.

