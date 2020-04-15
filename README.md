# Install Percona XtraDB Cluster on CentOS 7

In this article, we are installing Percona XtraDB Cluster on CentOS 7.
we are using just two nodes here, but the configurations do not vary, even if you are scaling to hundreds of nodes.

## Allow Percona XtraDB Cluster Service Ports in CentOS 7 Firewall:
Connect with pxc1.nazarbazi.ir using ssh as root user.
Percona XtraDB Cluster requires following service ports for communication, therefore, we are allowing these service ports in CentOS 7 firewall.
```
  firewall-cmd --permanent --add-port={3306,4444,4567,4568}/tcp<br>
  firewall-cmd --reload
```
Percona XtraDB Cluster is not fully compatible with SELinux, and it is recommended in PXC official documentation to put SELinux in permissive mode prior to installation.
```
  sed -i 's/^SELINUX=permissive$/SELINUX=disabled/' /etc/selinux/config
```

## Install Percona Yum Repository on CentOS 7:

Download and install rpm package for Percona yum repository as follows.
```
  yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
  yum makecache fast
```
Build cache for all yum repositories.

## Install Percona XtraDB Cluster using yum command. 
```
  yum install -y Percona-XtraDB-Cluster-57
```
Enable and start Percona database service.
```
  systemctl enable --now mysql.service
```
Percona installer generates a temporary password for root user in /var/log/mysqld.log file.
We can obtain this password using grep command.
```
  grep 'temporary password' /var/log/mysqld.log
```
Login to Percona instance using this temporary password.
```
  mysql -u root -p
```
Set a new password for root user.
```
  ALTER USER 'root'@'localhost' IDENTIFIED BY '123';
  flush privileges;
  exit
```
Stop Percona database service.
```
  systemctl stop mysql.service
```
### Repeat the above steps on pxc2.nazarbazi.ir.

## Configure PXC nodes for Write-set Replication:
Configure Percona XtraDB Cluster settings on pxc1.nazarbazi.ir.
```
  vim /etc/percona-xtradb-cluster.conf.d/wsrep.cnf
```
Find and set following directives therein.
```
  wsrep_cluster_address=gcomm://192.168.116.204,192.168.116.205

  wsrep_node_address=192.168.116.204

  wsrep_node_name=pxc1

  wsrep_sst_auth="sstuser:Mamad3r"
```

Configure Percona XtraDB Cluster settings on pxc2.nazarbazi.ir.
```
  vim /etc/percona-xtradb-cluster.conf.d/wsrep.cnf
```
Find and set following directives therein.
```
  wsrep_cluster_address=gcomm://192.168.116.204,192.168.116.205

  wsrep_node_address=192.168.116.205

  wsrep_node_name=pxc2

  wsrep_sst_auth="sstuser:Mamad3r"
```
## Bootstrap the first node:
We have configured both PXC nodes on CentOS 7.

Now its time to bootstap or initialize the first node of the Percona XtraDB Cluster.

First node of Percona XtraDB Cluster is the one that contains the data, that you want to replicate to other nodes.

Bootstrap the pxc1.nazarbazi.ir node using following command.

(Note: Please ensure that the mysql.service is stopped on all nodes.)
```
  systemctl start mysql@bootstrap.service
```
Connect with Percona database instance and check the configurations.
```
  mysql -u root -p

  show status like 'wsrep%';
```
Before adding any node to our cluster, we must create a user for SST (Snapshot State Transfer) for complete synchronization of new nodes with cluster.
```
  CREATE USER 'sstuser'@'%' IDENTIFIED BY 'Mamad3r';
  GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'sstuser'@'%';
  FLUSH PRIVILEGES;
  EXIT
```
## Adding Nodes to Percona XtraDB Cluster on CentOS 7:
Connect to pxc2.nazarbazi.ir using ssh as root user.

Start Percona service using systemctl command.
```
  systemctl start mysql.service
```
If our configurations are correct then the pxc2 node should receive the SST automatically.
```
  mysql -u root -p
  show status like 'wsrep%';
  show status like 'wsrep_cluster_size';
```
You can see that the wsrep_cluster_size is now 2, it shows that the pxc2 node has joined our Percona XtraDB Cluster.
## Verify Replication in our Percona XtraDB Cluster:
We can verify replication by manipulating data on one node and check if it replicated on the other node.
Connect with pxc2.nazarbazi.ir using ssh as root user.
Connect to Percona database instance and execute following commands.

```
  mysql> CREATE DATABASE RECIPES;
  Query OK, 1 row affected (0.06 sec)

  mysql> USE RECIPES;
  Database changed
  mysql> CREATE TABLE TAB1 (CONTACT_ID INT PRIMARY KEY, CONTACT_NAME VARCHAR(20));
  Query OK, 0 rows affected (0.05 sec)

  mysql> INSERT INTO TAB1 VALUES (1,'sadegh');
  Query OK, 1 row affected (0.04 sec)

  mysql>
  mysql> INSERT INTO TAB1 VALUES (2,'nix');
  Query OK, 1 row affected (0.00 sec)

  mysql> INSERT INTO TAB1 VALUES (3,'mamad');
  Query OK, 1 row affected (0.00 sec)
```

Connect with pxc1.nazarbazi.ir using ssh as root user.
Connect with Percona database instance and query the data that we have inserted on pxc2 node.
```
  mysql -u root -p
```
```
  mysql> USE RECIPES;
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A

  Database changed
  mysql> SELECT * FROM TAB1;
+------------+--------------+
| CONTACT_ID | CONTACT_NAME |
+------------+--------------+
|          1 | sadegh        |
|          2 | nix      |
|          3 | mamad
|
+------------+--------------+
3 rows in set (0.00 sec)
```
It shows that, our Percona XtraDB Cluster is working fine.
We have successfully installed and configured two node Percona XtraDB Cluster on CentOS 7.
