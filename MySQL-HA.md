#Creating High Available MySQL with HA Proxy

High Availability in MySQL is achieved through Clustering.

> Clustering connects multiple nodes, to form a single node. Virtual hosts, exchanges, users and permissions are mirrored across all nodes in a cluster. A client connecting to any node can see as a single cluster.

Types of High availability
1. Active/Passive - Stateless service would maintain a redundant instance that can be brought online when required. Requests may be handled using a virtual IP address to facilitate return to service with minimal reconfiguration required. Master – Salve Concept
2. Active/Active - Stateless service would maintain a redundant instance, and requests are load balanced using a virtual IP address and a load balancer such as HAProxy. Master – Master concept

Here we follow Active/Active High available MySQL.

####Set up:

Node1 – Node 2 – Connected in a same network

####Installation:

Install MySQL in all nodes.

```sh
yum install mysql-server
/sbin/service mysqld start
```
Then, run the following command:
```sh
sudo /usr/bin/mysql_secure_installation
```
Press enter to give no password for root when that program asks for it. To apply some reasonable security to your new MySQL server answer "yes" to all the questions that the program asks. In order, those questions enable you set the root password, remove anonymous users, disable remote root logins, delete the test database that the installer included, and then reload the privileges so that your changes will take effect.
To launch servie after machine starts
```sh
sudo chkconfig mysqld on
```
#####Allow access from other machines
Set the root password for MySQL
```sh
/usr/bin/mysqladmin -u root password 'new-pwd'
/usr/bin/mysqladmin -u root --password='new-pwd' -h hostname-of-your-server 'new-pwd'
```
If you have iptables enabled and want to connect to the MySQL database from another machine, you need to open a port in your server's firewall (the default port is 3306). 
If you do need to open a port, you can use the following rules in iptables to open port 3306:
```sh
iptables -I INPUT -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -I OUTPUT -p tcp --sport 3306 -m state --state ESTABLISHED -j ACCEPT
```
####Creating Cluster
In creating a cluster, we first start a single instance Node-1, which creates the cluster. The rest of the MySQL instances Node-2, then connect to that cluster:

Start on the first node having IP address 192.168.10.4 by executing the command:
```sh
service mysql start wsrep_cluster_address=gcomm://
```
_Connect to that cluster_ on the rest of the nodes by referencing the address of that node, as in:
```sh
service mysql start wsrep_cluster_address=gcomm://192.168.10.4
```
You also have the option to set the wsrep_cluster_address in the /etc/mysql/conf.d/wsrep.cnf file, or within the client itself.

Verify the clustering status, at Node-2, 
```mysql
mysql -u usrname -p pwd
SET GLOBAL wsrep_cluster_address='<cluster address string>';
SHOW STATUS LIKE 'wsrep%';
```
Next, we will add the MariaDB repo and install the MariaDB and Galera packages:
```sh
yum install python-software-properties
yum install mariadb-galera-server galera
```
Edit the configuration file
```sh
vi /etc/mysql/conf.d/cluster.cnf
[mysqld]
query_cache_size=0
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
query_cache_type=0
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_provider=/usr/lib/galera/libgalera_smm.so
#wsrep_provider_options="gcache.size=32G"

# Galera Cluster Configuration
wsrep_cluster_name="test_cluster"
wsrep_cluster_address="gcomm://192.168.10.4,192.168.10.5"

# Galera Synchronization Congifuration
wsrep_sst_method=rsync
#wsrep_sst_auth=user:pass

# Galera Node Configuration
wsrep_node_address="192.168.10.4"
wsrep_node_name="MySQL1"
```
Comment the bind address in MySQL config
```sh
vim /etc/mysql/my.cnf
#bind-address           = 127.0.0.1
```

Restart MySQL Service.
```sh
service mysql stop
service mysql start --wsrep-new-cluster
service mysql start
```
At Node1, configure MySQL as follows
```mysql
mysql -u root -p
grant all on *.* to root@'%' identified by 'password' with grant option;
insert into mysql.user (Host,User) values ('192.168.10.4','haproxy');
insert into mysql.user (Host,User) values ('192.168.10.5','haproxy');
flush privileges;
exit
```
####HA Proxy Integration
Add Galera to HA Proxy configuration
```sh
vim /etc/haproxy/haproxy.cfg
listen galera 192.168.10.1:3306
        balance source
        mode tcp
        option tcpka
        option mysql-check user haproxy
        server MySQL1 192.168.10.4:3306 check weight 1
        server MySQL2 192.168.10.5:3306 check weight 1
```
Now restart HAProxy
```sh
service haproxy reload
```
