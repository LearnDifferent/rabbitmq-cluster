# 部署 RabbitMQ 集群

**这里使用多个 Docker 容器在本地部署单机多实例 RabbitMQ 集群环境**

启动三个节点，一个主节点，一个从节点。

这里先启动一个节点 rabbit1，作为主节点（这里的主节点设置账号密码，其余节点不设置）：

```bash
docker run -d --hostname rabbit1host --name rabbit1 -p 15672:15672 -p 5672:5672 -e RABBITMQ_ERLANG_COOKIE='rabbitcookie' -e RABBITMQ_DEFAULT_USER=user -e RABBITMQ_DEFAULT_PASS=123 rabbitmq:3.8-management-alpine
```

然后启动节点 rabbit2，使用 `--link $需要连接的节点的name:$需要连接的节点的hostname` 来连接。（注意修改暴露的端口，避免端口冲突）：

```bash
docker run -d --hostname rabbit2host --name rabbit2 -p 15673:15672 -p 5673:5672 --link rabbit1:rabbit1host -e RABBITMQ_ERLANG_COOKIE='rabbitcookie' rabbitmq:3.8-management-alpine
```

最后，启动节点 rabbit3，这个节点需要同时连接之前的所有节点：

```bash
docker run -d --hostname rabbit3host --name rabbit3 -p 15674:15672 -p 5674:5672 --link rabbit1:rabbit1host --link rabbit2:rabbit2host -e RABBITMQ_ERLANG_COOKIE='rabbitcookie' rabbitmq:3.8-management-alpine
```

---

等所有节点（容器）启动完成后，重启每个节点（目的是让该节点离开其当前的集群），将从节点加入到主节点中。

对于主节点 rabbit1：

```bash
docker exec -it rabbit1 bash

# 使用 rabbitmqctl 命令，来停止当前节点
rabbitmqctl stop_app
# 重置节点的的数据，用于离开当前的集群
rabbitmqctl reset
# 现在启动该节点
rabbitmqctl start_app

exit
```

对于从节点 rabbit2:

```bash
docker exec -it rabbit2 bash
rabbitmqctl stop_app
rabbitmqctl reset
# 将当前的（rabbit2）节点，加入到 rabbit1 节点中（注意匹配的是 hostname）
# --ram：表示设置为内存节点（如果没有这个，就表示默认为磁盘节点）
rabbitmqctl join_cluster --ram rabbit@rabbit1host
rabbitmqctl start_app
exit
```

从节点 rabbit3，也要加入到主节点 rabbit1 中：

```bash
docker exec -it rabbit3 bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster --ram rabbit@rabbit1host
rabbitmqctl start_app
exit
```

> 加入完成后，可以进入任何一个节点（容器），使用 `rabbitmqctl cluster_status` 来查看集群信息

---

因为 RabbitMQ 使用的语言是 Erlang，所以 RabbitMQ 集群的节点间相互的通信需要 Erlang Cookie 的认证（相当于节点间交换信息的秘钥）。

> Erlang Cookie 的位置： `/var/lib/rabbitmq/.erlang.cookie` 

只要去集群中任意一个节点的 Erlang Cookie 所在的位置，将里面的内容拷贝。然后，去该集群的其他节点中，将拷贝的内容，覆盖（剪切）到该节点的 Erlang Cookie 中，就可以了。

注意：因为上面演示的时候，已经在启动节点容器的命令中，加入了 `RABBITMQ_ERLANG_COOKIE='rabbitcookie'` 参数，所以演示的时候，已经将所有节点的  Erlang Cookie，设置为了统一的 "rabbitcookie"。

参考资料：

- [使用docker搭建RabbitMQ集群](https://www.jianshu.com/p/d231844b9c46)
- [docker搭建RabbitMQ单机集群](https://www.jianshu.com/p/aa537ff043bc)
- [【集群运维篇】使用docker搭建RabbitMQ集群](https://blog.csdn.net/xia296/article/details/108395796)



***

# 管理 RabbitMQ 集群

下文摘抄自：[docker搭建RabbitMQ单机集群](https://www.jianshu.com/p/aa537ff043bc)

## 常用命令

**rabbitmqctl join_cluster {cluster_node} [–ram]**
 将节点加入指定集群中。在这个命令执行前需要停止RabbitMQ应用并重置节点。

**rabbitmqctl cluster_status**
 显示集群的状态。

**rabbitmqctl change_cluster_node_type {disc|ram}**
 修改集群节点的类型。在这个命令执行前需要停止RabbitMQ应用。

**rabbitmqctl forget_cluster_node [–offline]**
 将节点从集群中删除，允许离线执行。

**rabbitmqctl update_cluster_nodes {clusternode}**

在集群中的节点应用启动前咨询clusternode节点的最新信息，并更新相应的集群信息。这个和join_cluster不同，它不加入集群。考虑这样一种情况，节点A和节点B都在集群中，当节点A离线了，节点C又和节点B组成了一个集群，然后节点B又离开了集群，当A醒来的时候，它会尝试联系节点B，但是这样会失败，因为节点B已经不在集群中了。

**rabbitmqctl cancel_sync_queue [-p vhost] {queue}**
 取消队列queue同步镜像的操作。

**rabbitmqctl set_cluster_name {name}**
 设置集群名称。集群名称在客户端连接时会通报给客户端。Federation和Shovel插件也会有用到集群名称的地方。集群名称默认是集群中第一个节点的名称，通过这个命令可以重新设置。

## 管理节点

### 设置节点类型

如果你想更换节点类型可以通过命令修改，如下：

> rabbitmqctl stop_app
>
> rabbitmqctl change_cluster_node_type dist
>
> rabbitmqctl change_cluster_node_type ram
>
> rabbitmqctl start_app

### 移除节点

如果想要把节点从集群中移除，可使用如下命令实现：

> rabbitmqctl stop_app
>
> rabbitmqctl restart
>
> rabbitmqctl start_app

## 集群重启顺序

**集群重启的顺序是固定的，并且是相反的。** 如下所述：

- 启动顺序：磁盘节点 => 内存节点
- 关闭顺序：内存节点 => 磁盘节点

**最后关闭必须是磁盘节点**，不然可能回造成集群启动失败、数据丢失等异常情况

---

参考资料：

- [官网教程：Clustering Guide](https://rabbitmq.com/clustering.html)
- [RabbitMQ消息可靠投递和重复消费等问题解决方案](https://blog.csdn.net/qq_40837310/article/details/109033000)
