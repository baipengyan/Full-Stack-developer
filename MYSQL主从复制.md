



# MYSQL主从复制（docker）



## 主从规划

首先规划两个 MySQL 实例：

- 192.168.85.1:33061/mysql_master主机
- 192.168.85.1:33062/mysql_slave从机

当然大家可以准备多个从机，从机的配置步骤是一样的。



在 Docker 中创建两个 MySQL 实例的命令如下：

```
docker run --name mysql_master -p 33061:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:8.0.22 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci

docker run --name mysql_slave -p 33062:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:8.0.22 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```



创建完成后，通过 `docker ps` 命令可以查看 MySQL 实例的运行情况：





如此，表示 MySQL 实例已经在运行了。使用 Docker 配置 MySQL 主从最方便的莫过于配错了可以非常方便的从头开始。

接下来，我们就开始主从的配置。



## 配置主机

主机上的配置，主要是两个地方：

- 第一个是配置一个从机登录用户
- 第二个配置开启 binlog。

Docker 中创建的 MySQL 实例，默认只有一个用户，就是 root，这里我们需要进入到 MySQL 命令行，再给它分配一个用户。在宿主机中通过如下命令连接上主机：

```
mysql -u root -h 192.168.85.1 -P 33061 -p
```





输入密码后，进入到主机的命令行。然后给从机分配用户(因为我的宿主机上也安装了 MySQL，所以可以直接执行 mysql 命令，如果宿主机没有安装 MySQL，建议通过 `docker exec` 进入到 MySQL 容器中，然后执行如下命令)：

```
docker exec -it mysql_master bash
```



```
CREATE USER 'slave'@'%' IDENTIFIED BY 'slavepassword';	#创建用户

GRANT REPLICATION SLAVE ON *.* to 'slave'@'%';	#分配权限

ALTER USER 'slave'@'%' IDENTIFIED WITH mysql_native_password BY 'slavepassword';

flush privileges;   #刷新权限
```





这个表示从机一会使用 `slave/slavepassword` 来登录主机，`%` 表示这个账户可以从任意地址登录，也可以给一个固定的 `IP`，表示这个账户只能从某一个 IP 登录。









接下来开启 binlog。

binlog 的开启，需要修改 MySQL 的配置，因此，我们需要进入到容器内部去执行。

首先进入到容器内部：

```
docker exec -it mysql_master /bin/bash
```

然后找到 MySQL 配置文件的位置：

```
/etc/mysql/my.cnf
```

这就是 MySQL 的配置文件。我们要在这里进行修改操作。因为 MySQL 容器中，默认没有 VI 编辑器，安装费事，所以我们可以在宿主机中将配置文件写好，然后拷贝到 MySQL 容器中，覆盖原有配置。我们主要在该配置文件中添加如下内容：

```
log-bin=/var/lib/mysql/binlog
server-id=1
binlog-do-db = beloved
```

- 第一行表示配置 binlog 的位置，理论上 binlog 可以放在任意位置，但是该位置，MySQL 一定要有操作权限。
- server-id 表示集群中，每个实例的唯一标识符。
- bindlog-do-db 表示要同步的数据库有哪些。当从机连上主机后，并不是主机中的每一个库都要同步，这里表示配置哪些库需要同步。







配置完成后，保存退出。

接下来执行命令，将宿主机中的 my.cnf 拷贝到容器中：

```
docker cp ./my.cnf mysql_master:/etc/mysql/my.cnf
```

拷贝完成后，重启容器。

```
docker restart mysql_master
```

容器重启完成后，进入到主机的命令行中，查看配置是否成功：

```
SHOW MASTER STATUS;
```

File 和 Position 需要记着，这两个标记了二进制日志的起点位置，在从机的配置中将使用到这两个参数。

至此，主机的配置就算完成了。





## 配置从机

从机的配置比较简单，不用开启 binlog，也不用配置要同步的库，只需要在配置文件中，添加一个 server-id 即可。

这是从机的 my.cnf 配置：

```
server-id=2
```



配置完成后，一样拷贝到容器中。拷贝方式和主机一样：

```
docker cp ./my.cnf mysql_slave:/etc/mysql/my.cnf
```

配置完成后，重启从机容器：

```
docker restart mysql_slave
```

重启完成后，进入到 mysql_slave的命令行，执行如下命令，开启数据同步：

```
change master to master_host='192.168.85.1',master_port=33061,master_user='slave',master_password='slavepassword',master_log_file='binlog.000004',master_log_pos=368;
```

配置完成后，开启从机进程。在从机命令行执行如下命令：

```
start slave;
```



接下来，执行 show slave status；查看从机状态：

```
show slave status;
```

这里重点查看 Slave_IO_Running 和 Slave_SQL_Running ，这两个的值必须为 Yes。如果有一个的值不为 Yes，表示配置失败，一般情况下，配置失败，下面会有失败提示。

至此，我们的 MySQL 主从就算是配置成功了。



## 检验

配置成功之后，我们可以通过 Navicat 或者 DataGrip等工具连接上我们的两个 MySQL 实例，然后在主机中创建一个名为 beloved的库，你会发现从机中也会自动同步这个库。