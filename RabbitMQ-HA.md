#Creating High Available RabbitMQs with HA Proxy

High available queues is achieved through Clustering.
    
>Clustering connects multiple nodes, to form a single logical broker. Virtual hosts, exchanges, users and permissions are mirrored across all nodes in a cluster. A client connecting to any node can see all the queues in a cluster.

Types of Queue High availability<br>
1.	Active/Passive - Stateless service would maintain a redundant instance that can be brought online when required. Requests may be handled using a virtual IP address to facilitate return to service with minimal reconfiguration required.
Master – Salve Concept<br>
2.	Active/Active -  Stateless service would maintain a redundant instance, and requests are load balanced using a virtual IP address and a load balancer such as HAProxy. Master – Master concept

Here we follow Active/Active High available queues.

####Set up:  
> Node1 – Node 2 – Node 3 – Connected in a same network

###Installation
Install Rabbitmq in all nodes.
```sh
yum install rabbitmq-server
```
###Create Cluster
Stop rabbitmq service at all nodes
```sh
/etc/init.d/rabbitmq-server stop
```
At Node1,
Copy erlang cookie file and replace with other nodes
```sh
scp /var/lib/rabbitmq/.erlang.cookie root@node2:/var/lib/rabbitmq/.erlang.cookie
scp /var/lib/rabbitmq/.erlang.cookie root@node3:/var/lib/rabbitmq/.erlang.cookie
```
At Node2 and Node3,
Assign permission and access for rabbitmq user
```sh
chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
chmod 400 /var/lib/rabbitmq/.erlang.cookie
```
Start RabbitMQ Service at all nodes now
```sh
chkconfig rabbitmq-server on
/etc/init.d/rabbitmq-server start
```
Now there are three independent RabbitMQ brokers running. Next is to form them as a cluster and mirror the queues. To create the cluster, we have to join the first node - Node1 to form a cluster, since we copied the cookie file from Node1 to Node2 and Node3.

Login to Node2 and Node3, execute the following commands at both the nodes.
```sh
 rabbitmqctl stop_app
 rabbitmqctl join_cluster rabbit@node-01
 rabbitmqctl start_app
```

Check cluster status at all nodes, it has to be the same at all nodes.
```sh
rabbitmqctl cluster_status
```

Final step is to make the queues mirrored, Enabling mirrored queues allows producers and consumers to connect to any of the RabbitMQ brokers and access the same message queues.

```sh
rabbitmqctl set_policy HA '^(?!amq\.).*' '{"ha-mode": "all"}'
```

Now all the brokers, such as Node1, Node2 and Node3 shares the same message queues.
***
###Create HAProxy
For this setup, letus take Node1 for HAProxy installation
At Node1, do the following
```sh
 yum install haproxy
```
Now we have to create TCP proxy port 5670 on Node1 for HAProxy. Connections to this proxy will be distributed in a round-robin manner to the three RabbitMQ brokers, each listening on rabbitMQ port 5672.

Consider the node IP as follows
>Node1 - 192.168.10.1<br>
>Node2 - 192.168.10.2<br>
>Node3 - 192.168.10.3

Edit the HAProxy configuration file to update with TCP Proxy port as 5670
```sh
 vi /etc/haproxy/haproxy.cfg
```
haproxy.cfg content as follows
```sh
global
    daemon

defaults
    mode tcp
    maxconn 10000
    timeout connect 5s
    timeout client 100s
    timeout server 100s

listen rabbitmq 192.168.10.1:5670
    mode tcp
    balance roundrobin
    server node-01 192.168.10.1:5672 check inter 5s rise 2 fall 3
    server node-02 192.168.10.2:5672 check inter 5s rise 2 fall 3
    server node-03 192.168.10.3:5672 check inter 5s rise 2 fall 3
```
Now we are ready to start the HAProxy.
```sh
 service haproxy start
```
###Configure Openstack Services
Direct all the openstack services such as Nova, Neutron, Glance, Cinder etc to HAProxy node as **192.168.10.1:5670** for RabbitMQ access.

**Nova**:
```sh
openstack-config --set /etc/nova/nova.conf DEFAULT rabbit_host 192.168.10.1
openstack-config --set /etc/nova/nova.conf DEFAULT rabbit_port 5670
```
**Neutron**:
```sh
openstack-config --set /etc/neutron/neutron.conf DEFAULT rabbit_host 192.168.10.1
openstack-config --set /etc/neutron/neutron.conf DEFAULT rabbit_port 5670
```
**Nova**:
```sh
openstack-config --set /etc/nova/nova.conf DEFAULT rabbit_host 192.168.10.1
openstack-config --set /etc/nova/nova.conf DEFAULT rabbit_port 5670
```
**Glance**:
```sh
openstack-config --set /etc/glance/glance-api.conf DEFAULT rabbit_host 192.168.10.1
openstack-config --set /etc/glance/glance-api.conf DEFAULT rabbit_port 5670
```
**Cinder**:
```sh
openstack-config --set /etc/cinder/cinder.conf DEFAULT rabbit_host 192.168.10.1
openstack-config --set /etc/cinder/cinder.conf DEFAULT rabbit_port 5670
```
Last step to configure all RabbitMQ nodes to join the cluster on restart
Edit the configuration file at all nodes
```sh
vi /etc/rabbitmq/rabbitmq.config
```
Add this line to the file
```sh
[{rabbit,
  [{cluster_nodes, {['rabbit@node-01', 'rabbit@node-02', 'rabbit@node-03'], ram}}]}].
 ```

##Successfully achieved RabbitMQ HA with HAProxy
