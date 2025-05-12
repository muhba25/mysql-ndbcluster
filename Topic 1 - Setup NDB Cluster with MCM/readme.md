# Setup NDB Cluster with MySQL Cluster Manager (MCM)

## Download and Setup NDB Cluster with MySQL Cluster Manager
Download file mcm latest version and prepare the directory for data

```
mkdir /data/
mkdir /data01/
mkdir /data02/

mkdir /home/clustermanager/mcm8.0.42
unzip mcm8042.zip -d /home/clustermanager/mcm8.0.42/
cd /home/clustermanager/mcm8.0.42/
tar -zxvf mcm-8.0.42-cluster-8.0.42-linux-el8-x86-64bit.tar.gz

mv /home/clustermanager/mcm8.0.42/mcm-8.0.42-cluster-8.0.42-linux-el8-x86-64bit/* /data/
chown -R clustermanager:clustermanager /data
cd /data/

nano /etc/sudoers
clustermanager ALL=(ALL)       NOPASSWD: ALL

sudo ln -s /data/mcm8.0.42/bin/mcmd /bin/mcmd
sudo ln -s /data/mcm8.0.42/bin/mcm /bin/mcm
mcmd --version
mcm --version
ndb_mgmd --version
ndb_mgm --version
ndbd --version
ndbmtd --version

ATAU edit bash_profile

PATH=$PATH:$HOME/.local/bin:$HOME/bin
PATH=$PATH:/data/mcm8.0.42/bin:/data/cluster/bin
PATH=$PATH:$LD_LIBRARY_PATH:/usr/lib/:/usr/lib64/
export PATH

su - clustermanager
cd /data
mcmd &

```

## Stop and disable service firewalld and selinux

```
sestatus
vi /etc/selinux/config
SELINUX=disabled

systemctl stop firewalld
systemctl disable firewalld
```

## setup cluster
```
mcm> create site --hosts=192.168.1.11,192.168.1.12,192.168.1.13,192.168.1.14,192.168.1.15,192.168.1.16 sdc-mysql-site;
mcm> add package --basedir=/data/cluster 8.0.42;
mcm> create cluster --package=8.0.42 --processhosts=ndb_mgmd:145@192.168.1.11,ndb_mgmd:146@192.168.1.12,ndbmtd:1@192.168.1.13,ndbmtd:2@192.168.1.14,ndbmtd:3@192.168.1.15,ndbmtd:4@192.168.1.16,mysqld:147@192.168.1.11,mysqld:148@192.168.1.12,ndbapi@*,ndbapi@* dc01-mysql;

The NDB API is an object-oriented application programming interface for NDB Cluster that implements indexes, scans, transactions, and event handling.
```

## Set config for SQL Node
```
mcm> set port:mysqld=1707 dc01-mysql;
mcm> set default-storage-engine:mysqld=ndbcluster dc01-mysql;
mcm> set server-id:mysqld:147=21614511,server-id:mysqld:148=21614512 dc01-mysql;
mcm> set log_bin:mysqld="/log/geo-binlog-dc",max_allowed_packet:mysqld=1073741824,max_connect_errors:mysqld=4294967295,max_connections:mysqld=10000,slow_query_log:mysqld:147=1,slow_query_log_file:mysqld:147="/log/mysqld-slow.log" dc01-mysql;
mcm> set ndb_log_bin:mysqld=1 dc01-mysql;
mcm> set sql_mode:mysqld="ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION" dc01-mysql;

For bug
SET GLOBAL ndb_join_pushdown=OFF;
SET GLOBAL optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';
```

## Set config for Data Node
```
mcm> set DataDir:ndbmtd="/data01",BackupDataDir:ndbmtd="/backup",DataMemory:ndbmtd=400G,FileSystemPathDataFiles:ndbmtd="/data02",FileSystemPathUndoFiles:ndbmtd="/log",MaxBufferedEpochs:ndbmtd=10000,MaxNoOfAttributes:ndbmtd=30000,MaxNoOfConcurrentOperations:ndbmtd=81920,MaxNoOfConcurrentTransactions:ndbmtd=81920,MaxNoOfConcurrentScans:ndbmtd=500,MaxNoOfOrderedIndexes:ndbmtd=8192,MaxNoOfTables:ndbmtd=8192,MaxNoOfUniqueHashIndexes:ndbmtd=2048,NoOfFragmentLogFiles:ndbmtd=600,SharedGlobalMemory:ndbmtd=100G,TimeBetweenEpochsTimeout:ndbmtd=30000,TransactionDeadlockDetectionTimeout:ndbmtd=1200000,FragmentLogFileSize:ndbmtd=256M dc01-mysql;

mcm> set StopOnError:ndbmtd=false dc01-mysql;
mcm> set StopOnError:ndbmtd:3=false dc01-mysql;
mcm> set RedoOverCommitCounter:ndbmtd=10000,RedoOverCommitLimit:ndbmtd=10000 dc01-mysql;
mcm> set HeartbeatOrder:ndbmtd:1=10,HeartbeatOrder:ndbmtd:3=20,HeartbeatOrder:ndbmtd:3=30,HeartbeatOrder:ndbmtd:4=25 dc01-mysql;

mcm> set NoOfReplicas:ndbmtd=4 dc01-mysql;
mcm> set
ndb_cluster_connection_pool:mysqld=2,
ndb_cluster_connection_pool_nodeids:mysqld:147="147,149",
ndb_cluster_connection_pool_nodeids:mysqld:148="148,150"
 dc01-mysql;
```

## Set config for Management Node
```
OPTIONAL
set SendBufferMemory:ndbmtd+ndbmtd=500M dc01-mysql;
set ReceiveBufferMemory:ndbmtd+ndbmtd=500M dc01-mysql;
set SendBufferMemory:ndb_mgmd+ndbmtd=500M dc01-mysql;
set ReceiveBufferMemory:ndb_mgmd+ndbmtd=500M dc01-mysql;
set SendBufferMemory:mysqld+ndbmtd=500M dc01-mysql;
set ReceiveBufferMemory:mysqld+ndbmtd=500M dc01-mysql;
```

## Start Cluster & Check Cluster Status
```
mcm> start cluster dc01-mysql;
mcm> show status --process dc01-mysql;
```

## Additional Add Process & Setup Replication
```
// Additional how to add process

add process -R ndbapi@*,ndbapi@* dc01-mysql;
add process -R ndb_mgmd:146@192.168.1.12 dc01-mysql;
add process -R ndbmtd:1@192.168.1.13 dc01-mysql;
add process -R mysqld@192.168.1.17 dc01-mysql;

// Set Replica

On DC Node 1
mysql> create user 'repl'@'%' identified with mysql_native_password by 'testesreplika';
mysql> grant replication slave on *.* to 'repl'@'%';

On DRC Node 1
mysql> change master to master_host='10.216.145.11', master_user='repl', master_port=1707, master_password='woDRowg4nt3ng', MASTER_LOG_FILE = 'geo-binlog-dc.000001';
OR using MASTER_LOG_POS
mysql> change master to master_host='10.216.145.11', master_user='repl', master_port=1707, master_password='woDRowg4nt3ng', MASTER_LOG_FILE = 'geo-binlog-dc.000011', MASTER_LOG_POS = 94553;
mysql> start slave;
```