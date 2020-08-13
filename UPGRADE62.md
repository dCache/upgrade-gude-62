# The ultimate golden release upgrade guide

## Community effort to write an upgrade guide for dCache 6.2

How to get from dCache 5.2 to dCache 6.2

## Executive summary

### The highlights in 6.2 compared with 5.2

- bulk cache file cleanup on space pressure
- fifo/lifo capability of the flush queue
- TLS on p2p
- better support for distributed setups
- experimental internal message serialization format for better HPC support
- basic support for extended attributes
- full TLS support for all internal messages
- new telemetry service
- SciToken with XrootD
- Java 11
- systemd based packages

## Breaking changes

### Zookeeper

The dCache uses apache zookeeper for service discovery and coordination. Starting from version 6.2 the zookeeper 3.5.x is required. With this version jump, zookeeper supports TLS for communications between other zookeeper nodes as well as between zookeeper clients and server. This allows to make the whole dcache inter-component communication to be TLS protected, which is important for geo-distributed deployments over public networks.

The rpm packages with latest supported zookeeper server versions can be obtained from dcache.org yum repository:

```
[dcache-org-zk]
name=dCache.ORG zookeeper packages
baseurl=https://download.dcache.org/nexus/repository/zookeeper-rpms/el7/noarch
gpgcheck=0
enabled=1
```

### Java 11

Java 11 is the current LTS release of java runtime environment, that is required by dCache. In addition to language level changes for developers, the new JVM brings improvements for monitoring, integration with containers and debugging. For example, the jvm inspection tool [Flight Recorder](https://jdk.java.net/jmc/) is now available as a part of openJDK, the java thread names ca be shown by system tools, like `top -H <pid>` and with new JVM options `-XX:+PreserveFramePointer` the system profiler `perf` can collect and display method invocation statistics and call graphs.

>The all dCache releases are developed and tested with OpenJDK, which is license/subscription free. To use Oracle JVM you need `Java SE Subscriptions`.

### PostgreSQL

The PostgreSQL DB is an essential part dCache. It powers the namespace, AKA Chimera, PinManager, Srm, space manger and billing.

The minimal PostgeSQL version number supported by dCache is 9.5, which allows dCache use new SQL syntax to improve the DB operation performance.

### Integration with systemd

Starting from version 6.2 systemd is used to manage dCache service on the hosts. dCache uses systemd's generator functionality to create a service for each defined domain in the layout file. That's why, before starting the service all dynamic systemd units should be generated:

```
systemctl daemon-reload
```

> You need to regenerate dynamic units every time when new domains are added or removed as well as when systemd affected properties are modified. Those properties are:
> - dcache.user
> - dcache.java.options.extra
> - dcache.restart.delay
> - dcache.home
> - dcache.java.library.path

To inspect all generated units of dcache.target the `systemd list-dependencies` command can be used. For example:

```console-root
systemctl list-dependencies dcache.target
dcache.target
● ├─dcache@coreDomain.service
● ├─dcache@gplazmaDomain.service
● ├─dcache@namespaceDomain.service
● ├─dcache@nfsDomain.service
● ├─dcache@poolDomain.service
● └─dcache@poolmanagerDomain.service
```

> NOTE: the log files are no longer created in */var/log/dcache* and have to be accessed with through system journal, like `journalctl -u dcache@coreDomain.service`

To use the sysV-like `dcache [start|stop|status]` commands disable systemd for dCache by adding

```
dcache.systemd.strict=false
```
in *dcache.conf* or in the *layout* files.

For more information check the install chapter in the dcache book.

### Pool

The dCache pools stores the data in two places: a data file store and a complementary metadata store. The default metadata store was based on a pair of disk files for each data file. Over a decade a more effective BerkeleyDB-based metadata store was introduced and recommended for use in production. Now on, it becomes the default. To restore the old behavior add

```
pool.plugins.meta=org.dcache.pool.repository.meta.file.FileMetaDataRepository
```
into *dcache.conf* or *layout* files.

The BerkeleyDB-based metadata is enhanced with additional information which will be populated on pool start. Thus the very first start of the pools after upgrade to version 6.2 will take more time.

With introduction of `dCache zones`, the **zone** become a keyword and can't be used as a pool tag.

### Configuration properties

The following properties have changed their default values:

| property  | new value |
|:----------|-------:|
dcache.enable.authn.anonymous-fallback-on-failed-login | false
pool.plugins.meta | org.dcache.pool.repository.meta.db.BerkeleyDBMetaDataRepository
dcache.kafka.maximum-block | 60

## New services

### Telemetry

`Telememetry` is an optional service is added to collect statistics about running instances around the world. The telemetry cell has to be explicitly defined in a layout file as well as enabled:

```
[telemetryDomain]
[telemetryDomain/telemetry]
telemetry.cell.enable=true
telemetry.instance.site-name=dCache instance on \${host.name}
telemetry.instance.location.latitude=53.5772
telemetry.instance.location.longitude=9.8772
```

The dcache version, online capacity, site name and optional location will be sent to the stats.dcache.org collector.

The dCache.org kindly asks sites to enable telemetry service to let developers to see the deployments and version popularity.

> Though we don't collect any personal information, please consult you site security officer before enabling the service.
