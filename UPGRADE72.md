
# The ultimate golden release upgrade guide

## Community effort to write an upgrade guide for dCache 7.2

How to get from dCache 6.2 to dCache 7.2

## Executive summary

### The highlights in 7.2 compared with 6.2

-  quota system 
-  support for file labels
-  a dedicated scheduling strategy for bring-online requests in SRM-Manager


## Breaking changes


### Configuration properties
For File system statistics added functionality to cache total files and total space used on DB backend. 
The following two properties has been added:


| property  | new value |
|:----------|-------:|
pnfsmanager.fs-stat-cache.time | 3600
pnfsmanager.fs-stat-cache.time.unit | SECONDS


## New services

