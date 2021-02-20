# **第一节 Kubernetes上通过sts测试Redis Cluster集群**

## **1、Redis介绍**

Redis代表**Remote Dictionary Server** 是一种开源的内存中数据存储，通常用作数据库，缓存或消息代理。它可以存储和操作高级数据类型，例如列表，地图，集合和排序集合。由于Redis接受多种格式的密钥，因此可以在服务器上执行操作，从而减少了客户端的工作量。它仅将磁盘用于持久性，而将数据库完全保存在内存中。Redis是一种流行的数据存储解决方案，并被**GitHub，Pinterest，Snapchat，Twitter，StackOverflow，Flickr**等技术巨头所使用。

## **2、为什么使用Redis**

* 它的速度非常快。它是用ANSI C编写的，并且可以在POSIX系统上运行，例如Linux，Mac OS X和Solaris。
* Redis通常被排名为最流行的键/值数据库和最流行的与容器一起使用的NoSQL数据库。
* 其缓存解决方案减少了对云数据库后端的调用次数。
* 应用程序可以通过其客户端API库对其进行访问。
* 所有流行的编程语言都支持Redis。
* 它是开源且稳定的。

## **3、什么是Redis集群**

`Redis Cluster`是一组Redis实例，旨在通过对数据库进行分区来扩展数据库，从而使其更具弹性。

群集中的每个成员（无论是主副本还是辅助副本）都管理哈希槽的子集。如果主机无法访问，则其从机将升级为主机。**在由三个主节点组成的最小Redis群集中，每个主节点都有一个从节点（以实现最小的故障转移），每个主节点都分配有一个介于0到16,383之间的哈希槽范围。**

**节点A包含从0到5000的哈希槽，节点B从5001到10000，节点C从10001到16383。**群集内部的通信是通过内部总线进行的，使用协议传播有关群集的信息或发现新节点。

## **4、在kubernetes中部署redis集群**

在Kubernetes中部署Redis集群面临挑战，因为每个Redis实例都依赖于一个配置文件，该文件可以跟踪其他集群实例及其角色。为此，我们需要结合使用`Kubernetes StatefulSets`和`PersistentVolumes`。

### **4-1 部署Redis集群**

[部署及测试代码文件](https://github.com/Chao-Xi/jxredisbook/tree/master/docs/files/redis-sts)

创建statefulset类型资源 

**`redis-sts.yml`**

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster
data:
  update-node.sh: |
    #!/bin/sh
    REDIS_NODES="/data/nodes.conf"
    sed -i -e "/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/${POD_IP}/" ${REDIS_NODES}
    exec "$@"
  redis.conf: |+
    cluster-enabled yes
    cluster-require-full-coverage no
    cluster-node-timeout 15000
    cluster-config-file /data/nodes.conf
    cluster-migration-barrier 1
    appendonly yes
    protected-mode no
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:5.0.5-alpine
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
      # storageClassName: redis-cluster
      storageClassName: hostpath
```

* `cluster-node-timeout 15000`: [这个配置项设置了 Redis Cluster 中实例响应心跳消息的超时时间。](https://chao-xi.github.io/jxredisbook/chap7/1Redis_master_slave/#3-2-cluster-node-timeout)
* [` protected-mode no`](https://chao-xi.github.io/jxredisbook/chap7/1Redis_master_slave/#3-1-protected-mode): 当这个配置项设置为 yes 时，哨兵实例只能在部署的服务器本地进行访问。当设置为 no 时，其他服务器也可以访问这个哨兵实例。


```
$ kubectl apply -f redis-sts.yml
configmap/redis-cluster created
statefulset.apps/redis-cluster created

$ kubectl get pods -l app=redis-cluster
NAME              READY   STATUS    RESTARTS   AGE
redis-cluster-0   1/1     Running   0          88m
redis-cluster-1   1/1     Running   0          88m
redis-cluster-2   1/1     Running   0          174m
redis-cluster-3   1/1     Running   0          174m
redis-cluster-4   1/1     Running   0          174m
redis-cluster-5   1/1     Running   0          174m
```

**创建service**

**`redis-svc.yml`**

```
---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
spec:
  type: ClusterIP
  clusterIP: 10.96.0.106
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  selector:
    app: redis-cluster
```

```
$ kubectl apply -f redis-svc.yml
service/redis-cluster created

$ kubectl get svc 
NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)              AGE
kubernetes      ClusterIP   10.96.0.1     <none>        443/TCP              7d22h
redis-cluster   ClusterIP   10.96.0.106   <none>        6379/TCP,16379/TCP   6m9s
```

### **4-2 初始化redis cluster**

下一步是形成Redis集群。为此，我们运行以下命令并键入yes以接受配置。前三个节点成为主节点，后三个节点成为从节点。

```
$ kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 10.1.0.18:6379 to 10.1.0.14:6379
Adding replica 10.1.0.19:6379 to 10.1.0.15:6379
Adding replica 10.1.0.17:6379 to 10.1.0.16:6379
M: 61d39f27b3e2c7a11f54aed28e5c8096c6a63f02 10.1.0.14:6379
   slots:[0-5460] (5461 slots) master
M: 5fdd35b61af412cf8c8067764715acbd7c5adef1 10.1.0.15:6379
   slots:[5461-10922] (5462 slots) master
M: 6430f0c0de9f187ea0542ca84ac4ec2b9f920a2f 10.1.0.16:6379
   slots:[10923-16383] (5461 slots) master
S: fcf9deff9937b661135cedaa02f5cbb777fdd84e 10.1.0.17:6379
   replicates 6430f0c0de9f187ea0542ca84ac4ec2b9f920a2f
S: 06eaaac0487e4c2d3bed80782f6da5cf294b7381 10.1.0.18:6379
   replicates 61d39f27b3e2c7a11f54aed28e5c8096c6a63f02
S: 446198ba4c2a7a79cd37253a8fea076ddfa31f61 10.1.0.19:6379
   replicates 5fdd35b61af412cf8c8067764715acbd7c5adef1
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.....
>>> Performing Cluster Check (using node 10.1.0.14:6379)
M: 61d39f27b3e2c7a11f54aed28e5c8096c6a63f02 10.1.0.14:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 06eaaac0487e4c2d3bed80782f6da5cf294b7381 10.1.0.18:6379
   slots: (0 slots) slave
   replicates 61d39f27b3e2c7a11f54aed28e5c8096c6a63f02
M: 6430f0c0de9f187ea0542ca84ac4ec2b9f920a2f 10.1.0.16:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 446198ba4c2a7a79cd37253a8fea076ddfa31f61 10.1.0.19:6379
   slots: (0 slots) slave
   replicates 5fdd35b61af412cf8c8067764715acbd7c5adef1
M: 5fdd35b61af412cf8c8067764715acbd7c5adef1 10.1.0.15:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: fcf9deff9937b661135cedaa02f5cbb777fdd84e 10.1.0.17:6379
   slots: (0 slots) slave
   replicates 6430f0c0de9f187ea0542ca84ac4ec2b9f920a2f
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

### **4-3 验证集群**

```
$ kubectl exec -it redis-cluster-0 -- redis-cli cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:67
cluster_stats_messages_pong_sent:64
cluster_stats_messages_sent:131
cluster_stats_messages_ping_received:59
cluster_stats_messages_pong_received:67
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:131
```

```
$ for x in $(seq 0 5); do echo "redis-cluster-$x"; kubectl exec redis-cluster-$x -- redis-cli role; echo; done
redis-cluster-0
master
126
10.1.0.18
6379
126

redis-cluster-1
master
126
10.1.0.19
6379
126

redis-cluster-2
master
126
10.1.0.17
6379
126

redis-cluster-3
slave
10.1.0.16
6379
connected
126

redis-cluster-4
slave
10.1.0.14
6379
connected
126

redis-cluster-5
slave
10.1.0.15
6379
connected
126
```

## **5、测试集群**

我们想使用集群，然后模拟节点的故障。对于前一项任务，我们将部署一个简单的Python应用程序，而对于后者，我们将删除一个节点并观察集群行为。

```
$ kubectl exec -it redis-cluster-0 -- redis-cli 
127.0.0.1:6379> info
# Server
redis_version:5.0.5
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:4d072dc1c62d5672
redis_mode:cluster
os:Linux 4.19.121-linuxkit x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:atomic-builtin
gcc_version:8.3.0
process_id:1
run_id:5f57ff91b68a733e0e195a9ddd0ec79e31f80da9
tcp_port:6379
uptime_in_seconds:45890
uptime_in_days:0
hz:10
configured_hz:10
lru_clock:3175299
executable:/data/redis-server
config_file:/conf/redis.conf

# Clients
connected_clients:2
client_recent_max_input_buffer:2
client_recent_max_output_buffer:0
blocked_clients:0

# Memory
used_memory:2676408
used_memory_human:2.55M
used_memory_rss:5947392
used_memory_rss_human:5.67M
used_memory_peak:2676408
used_memory_peak_human:2.55M
used_memory_peak_perc:100.04%
used_memory_overhead:2595306
used_memory_startup:1463008
used_memory_dataset:81102
used_memory_dataset_perc:6.68%
allocator_allocated:3130336
allocator_active:3457024
allocator_resident:17768448
total_system_memory:8351621120
total_system_memory_human:7.78G
used_memory_lua:37888
used_memory_lua_human:37.00K
used_memory_scripts:0
used_memory_scripts_human:0B
number_of_cached_scripts:0
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
allocator_frag_ratio:1.10
allocator_frag_bytes:326688
allocator_rss_ratio:5.14
allocator_rss_bytes:14311424
rss_overhead_ratio:0.33
rss_overhead_bytes:-11821056
mem_fragmentation_ratio:2.26
mem_fragmentation_bytes:3312984
mem_not_counted_for_evict:112
mem_replication_backlog:1048576
mem_clients_slaves:16922
mem_clients_normal:66616
mem_aof_buffer:112
mem_allocator:jemalloc-5.1.0
active_defrag_running:0
lazyfree_pending_objects:0

# Persistence
loading:0
rdb_changes_since_last_save:1
rdb_bgsave_in_progress:0
rdb_last_save_time:1613742145
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:0
rdb_current_bgsave_time_sec:-1
rdb_last_cow_size:217088
aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_last_cow_size:0
aof_current_size:244
aof_base_size:188
aof_pending_rewrite:0
aof_buffer_length:0
aof_rewrite_buffer_length:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0

# Stats
total_connections_received:8
total_commands_processed:11979
instantaneous_ops_per_sec:1
total_net_input_bytes:447569
total_net_output_bytes:51635
instantaneous_input_kbps:0.03
instantaneous_output_kbps:7.01
rejected_connections:0
sync_full:1
sync_partial_ok:3
sync_partial_err:1
expired_keys:0
expired_stale_perc:0.00
expired_time_cap_reached_count:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:179
migrate_cached_sockets:0
slave_expires_tracked_keys:0
active_defrag_hits:0
active_defrag_misses:0
active_defrag_key_hits:0
active_defrag_key_misses:0

# Replication
role:master
connected_slaves:1
slave0:ip=10.1.0.18,port=6379,state=online,offset=16800,lag=1
master_replid:b7f58952b5529206bd0f637014de5ed204d6a22e
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:16800
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:16800

# CPU
used_cpu_sys:24.062666
used_cpu_user:8.067974
used_cpu_sys_children:0.001834
used_cpu_user_children:0.000000

# Cluster
cluster_enabled:1

# Keyspace
db0:keys=1,expires=0,avg_ttl=0
```

### **5-1 部署点击计数器应用**

我们将一个简单的应用程序部署到集群中，并在其前面放置一个负载平衡器。此应用程序的目的是在将计数器值作为HTTP响应返回之前，增加计数器并将其存储在Redis集群中。

**`app-deployment-service.yml`**

```
---
apiVersion: v1
kind: Service
metadata:
  name: hit-counter-lb
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
    targetPort: 5000
  selector:
      app: myapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hit-counter-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: calinrus/api-redis-ha:1.0
        ports:
        - containerPort: 5000
```

```
$ kubectl apply -f app-deployment-service.yml
service/hit-counter-lb created
deployment.apps/hit-counter-app created
```
```
$ kubectl get svc hit-counter-lb 
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
hit-counter-lb   ClusterIP   10.106.231.89   <none>        80/TCP    53m
```

在此过程中，如果我们继续加载页面，计数器将继续增加，并且在删除Pod之后，我们看到没有数据丢失。

### **5-2 启动Netbox检查网络**

在一个镜像中内置一些网络工具包，对我们排查工作会非常有帮助，比如在下面的简单服务中我们添加一些常用的网络工具包：`iproute2 net-tools ethtool`

`netbox-deployment.yaml`

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  labels:
    run: netbox
  name: netbox
spec:
  replicas: 2
  selector:
    matchLabels:
      run: netbox
  template:
    metadata:
      labels:
        run: netbox
    spec:
      nodeSelector:
        type: other      
      containers:
      - image: quay.io/gravitational/netbox:latest
        imagePullPolicy: Always
        name: netbox
      securityContext:
        runAsUser: 0
      terminationGracePeriodSeconds: 30
```

```
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
hit-counter-app-6987969599-64ndt   1/1     Running   0          13h
netbox-6b9767f659-9kbc6            1/1     Running   0          12h
```

```
kubectl exec -it netbox-6b9767f659-9kbc6 bash

root@netbox-6b9767f659-9kbc6:/# curl 10.106.231.89
I have been hit 1 times since deployment.
root@netbox-6b9767f659-9kbc6:/# curl 10.106.231.89
I have been hit 2 times since deployment.
root@netbox-6b9767f659-9kbc6:/# curl 10.106.231.89
I have been hit 3 times since deployment.
root@netbox-6b9767f659-9kbc6:/# curl 10.106.231.89
I have been hit 4 times since deployment.
root@netbox-6b9767f659-9kbc6:/# curl 10.106.231.89
I have been hit 5 times since deployment.
root@netbox-6b9767f659-9kbc6:/# 

kubectl delete pods redis-cluster-0
pod "redis-cluster-0" deleted

$ kubectl delete pods redis-cluster-1
pod "redis-cluster-1" deleted

$ kubectl exec -it netbox-6b9767f659-9kbc6 bash 
root@netbox-6b9767f659-9kbc6:/# curl 10.106.231.89
I have been hit 6 times since deployment.
```



