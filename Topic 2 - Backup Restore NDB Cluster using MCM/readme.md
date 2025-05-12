# BACKUP-RESTORE

verify NDB API process Available on Cluster
```
mcm> show status -r dc01-mysql;
+--------+----------+--------------+---------+-----------+---------+
| NodeId | Process  | Host         | Status  | Nodegroup | Package |
+--------+----------+--------------+---------+-----------+---------+
| 145    | ndb_mgmd | 192.168.1.11 | running |           | 8.0.42  |
| 146    | ndb_mgmd | 192.168.1.12 | running |           | 8.0.42  |
| 1      | ndbmtd   | 192.168.1.13 | running | 0         | 8.0.42  |
| 2      | ndbmtd   | 192.168.1.14 | running | 0         | 8.0.42  |
| 3      | ndbmtd   | 192.168.1.15 | running | 0         | 8.0.42  |
| 4      | ndbmtd   | 192.168.1.16 | running | 0         | 8.0.42  |
| 147    | mysqld   | 192.168.1.11 | running |           | 8.0.42  |
| 148    | mysqld   | 192.168.1.12 | running |           | 8.0.42  |
| ```**149**```    | ```**ndbapi**```   | *            | ```added```   |           |         |
| ```**150**```    | ```**ndbapi**```   | *            | ```added```   |           |         |
+--------+----------+--------------+---------+-----------+---------+

```

##  Backup Database
```
// Cek BackupDatadir atau set ketika setup Datanode

mcm> get *backup*:ndbmtd dc01-mysql;
example output
+---------------+---------+----------+---------+----------+---------+---------+---------+
| Name          | Value   | Process1 | NodeId1 | Process2 | NodeId2 | Level   | Comment |
+---------------+---------+----------+---------+----------+---------+---------+---------+
| BackupDataDir | /backup | ndbmtd   | 1       |          |         | Process |         |
| BackupDataDir | /backup | ndbmtd   | 2       |          |         | Process |         |
| BackupDataDir | /backup | ndbmtd   | 3       |          |         | Process |         |
| BackupDataDir | /backup | ndbmtd   | 4       |          |         | Process |         |
+---------------+---------+----------+---------+----------+---------+---------+---------+

// Run Backup 
mcm> backup cluster --backupid=1 --background dc01-mysql;

// Monitor the backup process using the following:
mcm> show status --progress mycluster;

// list all backups for mycluster
mcm> list backups mycluster;
example output 
+----------+--------+--------------+----------------------+-------+---------+
| BackupId | NodeId | Host         | Timestamp            | Parts | Comment |
+----------+--------+--------------+----------------------+-------+---------+
| 2        | 1      | 192.168.1.13 | 2025-04-26 13:50:05Z | 1     |         |
| 2        | 2      | 192.168.1.14 | 2025-04-26 13:49:01Z | 1     |         |
| 2        | 3      | 192.168.1.15 | 2025-04-26 14:01:06Z | 1     |         |
| 2        | 4      | 192.168.1.16 | 2025-04-26 14:05:47Z | 1     |         |
+----------+--------+--------------+----------------------+-------+---------+

mcm> exit

// On shell, check the backup files from backupdatadir file location
ls -lthr /backup/BACKUP/BACKUP-2/

```

## Stop and Start Cluster with Initial
```
mcm> stop cluster --background mycluster;
mcm> show status --progress mycluster;

mcm> \! rm -Rf /home/opc/mysql-cluster/mcm-data/clusters/mycluster/146/data/*
mcm> \! rm -Rf /home/opc/mysql-cluster/mcm-data/clusters/mycluster/147/data/*

mcm> start cluster --initial --background mycluster;
mcm> show status --progress mycluster;
mcm> show status -r mycluster;
mcm> exit;
```

## check content of database
```
mysql -uroot -h::1 -e "show databases"

mysql -uroot -h::1 -P3307 -e "show databases"
```

## Restore from backup using mcm
```
mcm
mcm> list backups mycluster;
mcm> restore cluster --backupid=1 --background mycluster;
mcm> show status --progress mycluster;
mcm> exit;

// Check Data from SQL node
mysql -uroot -h::1 -e "show databases"
mysql -uroot -h::1 -P3307 -e "show databases"
```

## Partial restore using mcm
```
// Delete Table
mysql -uroot -h::1 -e "drop table world_x.city"
// Restore table using mcm
mcm
mcm> restore cluster --backupid=1 --include-tables=schemadb.tablename mycluster; 
mcm> exit
```