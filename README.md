# yugabytedb-logs

## Yugabyte recovery after cluster outage

The logs logs.tar.gz show how yuagbyte restarts a couple of times before recovering from a full outage.
The tserver folders contain each a splitted tar.gz file of the tablet servers data.
The data is split into 100 MB chunks which can be put together to a tar.gz file using `cat tserver-X/* > tserver-X.tar.gz`.

It seems that the restarts happen only if encryption at rest is enabled and the flags `durable_wal_write` and `require_durable_wal` are set to true.

The encryption key id is yb-key-1674114520 and the file yb-key-1674114520.base64 contains the base64 encoded key.

The cluster outage was simulated by killing all tablet servers at the same time.
The following logs show how yugabyte recovers from the outage.
Multiple restarts of the tablet servers are needed until the tablet servers fully recover.


### To reproduce this scenario

Create a k8 cluster (e.g. minikube).
Use the yugabyte helm chart https://artifacthub.io/packages/helm/yugabyte/yugabyte version 2.14.3 and the image 2.14.3.1-b1.
Enable `durable_wal_write` and `require_durable_wal_write`.
Start a minimal yugabyte cluster (3 master, 3 tablets).

Simulate some client activity:

ycqlsh:
```
create keyspace ks;
create table ks.test1(key text primary key, value text);
create table ks.test2(key text primary key, value text);
create table ks.test3(key text primary key, value text);
create table ks.test4(key text primary key, value text);
```

generate some scripts:
```
for i in `seq 1 1000`; do echo "insert into ks.test1(key, value) values('k$i', 'v1');"; done > insert1.cql
for i in `seq 1 1000`; do echo "insert into ks.test2(key, value) values('k$i', 'v1');"; done > insert2.cql
for i in `seq 1 1000`; do echo "insert into ks.test3(key, value) values('k$i', 'v1');"; done > insert3.cql
for i in `seq 1 1000`; do echo "insert into ks.test4(key, value) values('k$i', 'v1');"; done > insert4.cql
```

run the scripts:
```
ycqlsh -f insert1.cql &
ycqlsh -f insert2.cql &
ycqlsh -f insert3.cql &
ycqlsh -f insert4.cql &
```

kill all tablet servers:
```
kubectl exec yb-tserver-0 -c yb-tserver -- kill 1 
kubectl exec yb-tserver-1 -c yb-tserver -- kill 1 
kubectl exec yb-tserver-2 -c yb-tserver -- kill 1 
```

watch the recovery process
