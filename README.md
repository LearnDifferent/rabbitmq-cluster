# 使用多个 Docker 容器在本地单机部署多实例 RabbitMQ 集群环境

启动三个节点，一个主节点，一个从节点。

这里先启动一个节点 rabbit1，作为主节点（主节点设置账号密码，其余节点不设置）：

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

