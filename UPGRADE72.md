
# The ultimate golden release upgrade guide

## Community effort to write an upgrade guide for dCache 7.2

How to get from dCache 6.2 to dCache 7.2

## Executive summary

### The highlights in 7.2 compared with 6.2

-  quota system 
-  support for file labels
-  a dedicated scheduling strategy for bring-online requests in SRM-Manager
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

## New services

