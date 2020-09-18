# Mysql HA


## HA 方案描述

1. Mysql 主从复制的集群 Master - Slave.

2. 唯一的主节点.

3. 多个从节点.

4. 从节点水平扩展.

---

> 基于 Statefulset 与 Percona-Xtrabackup 实现 Mysql HA.

## StatefulSet 

* `Pod` 会被顺序部署和顺序终结, 会按照 0-N 顺序. 如: mysql-0 , mysql-1

* `Pod` 具有唯一的名称，而且在重启后会保持不变, 基于 Pod 名称 + Headless 服务名称. 可以让 `Pod` 具有独立的网络地址.

* `Pod` 具有稳定的持久化存储, StatefulSet中的每个 Pod 可以有其自己独立的 PersistentVolumeClaim 对象.

---

## Xtrabackup 

* Percona-xtrabackup 是 Percona 公司开发的一个用于 MySQL数据库物理热备的备份工具, 支持MySQL、Percona Server和MariaDB, 开源免费，是目前较为受欢迎的主流备份工具. 目前Xtrabackup 只能备份 innoDB 和 xtraDB 两种数据引擎的表，而不能备份MyISAM数据表.


* xtrabackup 主要特点

  * 备份速度快，物理备份可靠.
  * 备份过程不会打断正在执行的事务（无需锁表）.
  * 能够基于压缩等功能节约磁盘空间和流量.
  * 自动备份校验.
  * 还原速度快.

---

## Mysql 主从实现流程


* 创建 基于 statefulset 的两个 Service 服务.

  * Headless Service 服务 - 用于实现 `Pod` 的唯一网络名称, 以及 Master 写节点地址.
  * 普通 Service 服务 - 用于 Slave 读节点地址.

* 初始化 Mysql 配置文件. 

  * 根据 statefulset 生成 Pod 的名称, 区分 master, slave 配置文件,以及用于主从复制的 `Server-id` 标识.

* 初始化 Master Mysql-0 数据库, 生成数据库文件. 此时  Master 节点已经准备就绪.

* Slave Mysql-1 节点 初始化阶段

  * 通过 nact 命令从 Pod 名称 - 1 的容器中拷贝 数据文件到容器里.
  * 将拷贝的数据文件 通过  xbstream 命令打包成 xtrabackup 的数据恢复文件.
  * 通过 `xtrabackup --prepare` 全量恢复数据文件.

* 启动 Slave Mysql-1 数据库节点.

* 配置 主从复制.

  * 通过 数据文件中的 `xtrabackup_slave_info` 以及 `xtrabackup_binlog_info` 文件 获取 `gtid_purged` 记录.

  * 拼接 主从配置 语句, 并写入到 `change_master_to.sql.in` 文件中. 

  * 通过 mysql -e 执行 文件中拼接好的 主从复制 SQL 语句. 

  * 使用 ncat 命令创建一个 3307 端口的连接, 接收其他节点调用时创建 xtrabackup 备份文件.并将备份文件传输到调用节点.

  * 此时 slave 节点准备就绪.

---

## 部署 Mysql-ha


* `ops-kubeconfig` 项目中.

  * 进入 projects 目录

  * 复制一份项目, 如: 

	* cp -r mysql/demo-mysql-ha/  mysql/zabbix-mysql

* 进入刚才复制的项目

* 修改相关文件的配置信息. 

  * Chart.yaml 文件 注意修改如下字段:

    * 修改  name: zabbix-mysql

  * values.{对应环境名称}.yaml 文件  注意修改如下字段:

    * env.MYSQL_DATABASE  初始化创建 的数据库.

    * env.MYSQL_USER  初始化创建 MYSQL_DATABASE 数据库的关联 用户.

    * resources  限制 Mysql 运行的 cpu 以及 内存.

    * persistence 是否开启挂载 文件系统, false 为 emptyDir .  

    * config.ini 文件 修改如下字段:
		
      * namespace=zabbix  修改为与项目相同的 namespace .
---

* `ops-env` 项目中

  * 复制一份相同名称的项目配置信息. 如: 
    * cp -r demo-mysql-ha  zabbix-mysql

  * 进入刚才复制的项目 

  * 修改配置相关的信息. 

    * development/config/mysql.conf  -- Mysql 相关配置信息.

	* development/env.enc.yml  --  Mysql 密码信息的加密文件. 需要自行重新创建. env.yml 包含如下信息: 

	  * `MYSQL_ROOT_PASSWORD: 123445`    # mysql root 账户的密码
	  * `MYSQL_PASSWORD: mysqldemo`      # 如上配置 env.MYSQL_USER 账户的密码
	  * `TZ: UTC`                        # 时区

  * 提交 pr 到 gitlab , 合并到 master

---


### 数据库调用

1. 程序调用写数据使用 Master pod 名称 + headless service 名称 . 如:  `mysql-0.mysql-headless` .

2. 程序读数据时 使用 普通的 service 名称. 如: `mysql-read` . 注: 此名称会包含 Master 节点. 如需调用只读可使用 `mysql-1.mysql-headless`.


---

### 扩展节点

1. 扩展从节点的数量

  1. 修改 helm 模板文件 shared.yaml 中的 replicaCount 数量.
  2. 使用 kubectl scale statefulset mysql --replicas=5 来直接扩展.

---



### 验证数据库


* 查看已创建数据库


```
root@shell:/srv/ops-kubeconfig# kubectl get pods -n mysql-ha
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   3/3     Running   0          21m
mysql-1   3/3     Running   0          17m
mysql-2   3/3     Running   0          13m
mysql-3   3/3     Running   0          8m24s
mysql-4   3/3     Running   0          4m11s

```


* 验证 数据库 创建



```
root@shell:/srv/ops-kubeconfig# kubectl exec -it mysql-0 -n mysql-ha -c mysql -- mysql -uroot -pmysqldev -e "create database test_demo;"


# 查看 Mysql-0 数据库
root@shell:/srv/ops-kubeconfig# kubectl exec -it mysql-0 -n mysql-ha -c mysql -- mysql -uroot -pmysqldev -e "show databases;"

+------------------------+
| Database               |
+------------------------+
| information_schema     |
| mysql                  |
| mysql_demo             |
| performance_schema     |
| sys                    |
| test_demo              |
| xtrabackup_backupfiles |
+------------------------+



# 查看 Mysql-4 数据库

root@shell:/srv/ops-kubeconfig# kubectl exec -it mysql-4 -n mysql-ha -c mysql -- mysql -uroot -pmysqldev -e "show databases;"
+------------------------+
| Database               |
+------------------------+
| information_schema     |
| mysql                  |
| mysql_demo             |
| performance_schema     |
| sys                    |
| test_demo              |
| xtrabackup_backupfiles |
+------------------------+



# 查看 binlog 日志

root@shell:/srv/ops-kubeconfig# kubectl exec -it mysql-0 -n mysql-ha -c mysql -- mysql -uroot -pmysqldev -e "show master logs;"
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |   8191811 |
| mysql-bin.000003 |       368 |
+------------------+-----------+


root@shell:/srv/ops-kubeconfig# kubectl exec -it mysql-0 -n mysql-ha -c mysql -- mysql -uroot -pmysqldev -e "show binlog events in 'mysql-bin.000003';"
+------------------+-----+----------------+-----------+-------------+--------------------------------------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                                               |
+------------------+-----+----------------+-----------+-------------+--------------------------------------------------------------------+
| mysql-bin.000003 |   4 | Format_desc    |       100 |         123 | Server ver: 5.7.31-log, Binlog ver: 4                              |
| mysql-bin.000003 | 123 | Previous_gtids |       100 |         194 | e9231f8c-f8a7-11ea-8bac-26baa7120028:1-9                           |
| mysql-bin.000003 | 194 | Gtid           |       100 |         259 | SET @@SESSION.GTID_NEXT= 'e9231f8c-f8a7-11ea-8bac-26baa7120028:10' |
| mysql-bin.000003 | 259 | Query          |       100 |         368 | create database test_demo                                          |
+------------------+-----+----------------+-----------+-------------+--------------------------------------------------------------------+


```

* 查看从节点情况

```
root@shell:/srv/ops-kubeconfig# kubectl exec -it mysql-1 -n mysql-ha -c mysql -- mysql -uroot -pmysqldev -e "show slave status\G;"
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-0.mysql-headless
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 368
               Relay_Log_File: relay.000002
                Relay_Log_Pos: 541
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 368
              Relay_Log_Space: 738
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 100
                  Master_UUID: e9231f8c-f8a7-11ea-8bac-26baa7120028
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: e9231f8c-f8a7-11ea-8bac-26baa7120028:10
            Executed_Gtid_Set: e9231f8c-f8a7-11ea-8bac-26baa7120028:1-10
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 

```



* 验证从节点创建数据


```
root@shell:/srv/ops-kubeconfig# kubectl exec -it mysql-1 -n mysql-ha -c mysql -- mysql -uroot -pmysqldev -e "create database demo;"

ERROR 1290 (HY000) at line 1: The MySQL server is running with the --super-read-only option so it cannot execute this statement
command terminated with exit code 1

```


---





### 破坏性测试


* 测试环境说明

  * statefulset - mysql - replicas = 5

  * storageClass: gfs-delete  容量 20G

  * 测试数据:  689M



> 模拟 Mysql 主节点 宕机
 

```
# 统计数据库大小

root@shell:/srv/ops-kubeconfig# kubectl exec -it mysql-0 -n mysql-ha -c mysql -- mysql -uroot -pmysqldev -e "use information_schema; select concat(round(sum(DATA_LENGTH/1024/1024), 2), 'MB') as data from TABLES;"

+----------+
| data     |
+----------+
| 966.89MB |
+----------+


```




```
# 查看 Mysql 节点
root@shell:/srv/ops-kubeconfig# kubectl get pods -n mysql-ha
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   3/3     Running   0          111m
mysql-1   3/3     Running   0          45m
mysql-2   3/3     Running   0          12m
mysql-3   3/3     Running   0          52m
mysql-4   3/3     Running   0          39m


# 删除 主节点 模拟宕机
root@shell:/srv/ops-kubeconfig# kubectl delete pods mysql-0 -n mysql-ha
pod "mysql-0" deleted

```


* 查看主从状态情况

  * Last_IO_Error: error reconnecting to master 'root@mysql-0.mysql-headless:3306' - retry-time: 10  retries: 7

```
kubectl exec -it mysql-1 -n mysql-ha -c mysql -- mysql -uroot -pmysqldev -e "show slave status\G;"

*************************** 1. row ***************************
               Slave_IO_State: Reconnecting after a failed master event read
                  Master_Host: mysql-0.mysql-headless
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 159626166
               Relay_Log_File: relay.000009
                Relay_Log_Pos: 159626379
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Connecting
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 159626166
              Relay_Log_Space: 159626663
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 2005
                Last_IO_Error: error reconnecting to master 'root@mysql-0.mysql-headless:3306' - retry-time: 10  retries: 7
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 100
                  Master_UUID: e9231f8c-f8a7-11ea-8bac-26baa7120028
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 200917 07:27:29
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: e9231f8c-f8a7-11ea-8bac-26baa7120028:43-1393
            Executed_Gtid_Set: e9231f8c-f8a7-11ea-8bac-26baa7120028:1-1393
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version:

```

* 等待 Master 节点重建

```
root@shell:/srv/ops-kubeconfig# kubectl get pods -n mysql-ha -w
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   0/3     Running   0          27s
mysql-1   3/3     Running   0          46m
mysql-2   3/3     Running   0          13m
mysql-3   3/3     Running   0          53m
mysql-4   3/3     Running   0          40m
mysql-0   1/3     Running   0          40s
mysql-0   2/3     Running   0          70s
mysql-0   3/3     Running   0          73s

```

* 再次查看 从节点的 主从状态  已经恢复正常

```
root@shell:/srv/ops-kubeconfig# kubectl exec -it mysql-1 -n mysql-ha -c mysql -- mysql -uroot -pmysqldev -e "show slave status\G;"

*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-0.mysql-headless
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 194
               Relay_Log_File: relay.000010
                Relay_Log_Pos: 407
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 194
              Relay_Log_Space: 159626829
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 100
                  Master_UUID: e9231f8c-f8a7-11ea-8bac-26baa7120028
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: e9231f8c-f8a7-11ea-8bac-26baa7120028:43-1393
            Executed_Gtid_Set: e9231f8c-f8a7-11ea-8bac-26baa7120028:1-1393
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 

```




## 维护说明

> 批量插入测试数据

  * sh create_data.sh 插入数量 密码  如: ./create_data.sh 100 mysqldev

```
#!/bin/bash

:<<!
create database test_demo;

CREATE TABLE `test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(64) NOT NULL,
  `age` int(6) NOT NULL,
  `createTime` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

!


i=1;
MAX_INSERT_ROW_COUNT=$1;
MYSQL_PASSWORD=$2;
while [ $i -le $MAX_INSERT_ROW_COUNT ]
do
    mysql -uroot -p$MYSQL_PASSWORD test_demo -e "insert into test (name,age,createTime) values ('HELLO$i',$i % 99,NOW());"
    d=$(date +%M-%d\ %H\:%m\:%S)
    echo "INSERT HELLO $i @@ $d"
    i=$(($i+1))
    sleep 0.05
done
exit 0


```




> 统计数据库大小 命令


```
1、进去指定schema 数据库：
use information_schema;

2、查询所有数据的大小 :

select concat(round(sum(DATA_LENGTH/1024/1024), 2), 'MB') as data from TABLES;

3、查看指定数据库实例的大小:

select concat(round(sum(DATA_LENGTH/1024/1024), 2), 'MB') as data from TABLES where table_schema='db_name';

4、查看指定数据库的表的大小: 

select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from TABLES where table_schema='db_name' and table_name='tb_name';


5、 查看每一个数据库中的每一个表的大小

select table_name,table_rows,data_Length,index_length from TABLES;
```



---


> 由于异常断电造成的 `relay log` 损坏 同步失败

* 报错如下:

  * Last_SQL_Error: Relay log read failure: Could not parse relay log event entry. The possible reasons are: the master's binary log is corrupted (you can check this by running 'mysqlbinlog' on the binary log), the slave's relay log is corrupted (you can check this by running 'mysqlbinlog' on the relay log), a network problem, or a bug in the master's or slave's MySQL code. If you want to check the master's binary log or slave's relay log, you will be able to know their names by issuing 'SHOW SLAVE STATUS' on this slave.

  * 解决方式  

```
# 1. 登录同步错误的节点

# 2. 停止 主从同步

stop slave; 

# 3. 清除 relay log 日志  reset slave  会清除master.info、relay-log.info以及所有relay log文件.

reset slave;

# 4. 删除当前 pods 重建

kubectl delete pods mysql-2 -n mysql-ha


# 5. 重新启动主从同步

start slave;
```

