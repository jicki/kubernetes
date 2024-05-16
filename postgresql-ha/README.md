# 容器化 HA Postgresql 方案

## HA 方案描述

---

* Mysql 主从复制的集群 Master - Slave.

* 唯一的主节点.

* 多个从节点.

* 从节点水平扩展.

---

## 数据库调用

* 程序调用写数据使用 Master pod 名称 + headless service 名称 . 如: postgresql-0.postgresql-headless 

* 程序读数据时 使用 普通的 service 名称. 如: postgresql-read . 注: 此名称会包含 Master 节点. 如需调用只读可使用 postgresql-1.postgresql-headless


## 扩展节点

* 扩展从节点的数量
    
  * 使用 kubectl scale statefulset postgresql --replicas=5 来直接扩展.



## 验证集群

* 查看已创建数据库

```
root@shell:/srv/ops-kubeconfig# kubectl get pods -n pgsql
NAME           READY   STATUS    RESTARTS   AGE
postgresql-0   2/2     Running   0          6m7s
postgresql-1   2/2     Running   0          10m
postgresql-2   2/2     Running   0          60m
```


* 验证数据库主从复制

```
postgres@postgresql-0:/$ psql 
psql (9.6.16)
Type "help" for help.

postgres=# select pid,usesysid,usename,client_addr,state,sync_state from pg_stat_replication;
 pid | usesysid | usename  |   client_addr   |   state   | sync_state 
-----+----------+----------+-----------------+-----------+------------
  29 |       10 | postgres | 172.110.101.123 | streaming | async
  30 |       10 | postgres | 172.110.101.125 | streaming | async
(2 rows)

```

---

* 创建数据库 以及 插入数据 主节点

```
postgres@postgresql-0:/$ psql 
psql (9.6.16)
Type "help" for help.

postgres=# create table test2(id int primary key, name varchar(20));
CREATE TABLE


postgres=# insert into test2 values(1,'aaa'),(2,'abc');
INSERT 0 2
```


* 验证数据同步

```
postgres@postgresql-1:/$ psql 
psql (9.6.16)
Type "help" for help.

postgres=# select * from test2;
 id | name 
----+------
  1 | aaa
  2 | abc
(2 rows)
```

---


* 从库插入数据

```
postgres@postgresql-2:/$ psql 
psql (9.6.16)
Type "help" for help.

postgres=# insert into test2 values(7,'aaa'),(8,'abc');
ERROR:  cannot execute INSERT in a read-only transaction

```



## 破坏性测试

* 操作流程
  
  * 持续往主库写入数据
  
  * 删除 从库节点, 等待恢复, 查看是否数据仍然会同步.
  
  * 删除 主节点, 等待恢复, 再插入数据, 查看从库是否仍然会同步.


---

* 批量插入数据脚本

```
#!/bin/sh

:<<!
创建数据库:

  CREATE DATABASE authdb;

切换数据库:

  \c authdb;

创建表:

CREATE TABLE server_auth_info (
id int PRIMARY KEY,
server_name char(50),
auth_code char(50)
);  

!

# 初始化参数
i=1
MAX_INSERT_ROW_COUNT=$1
db_name="authdb"
table_name="server_auth_info"


# 数据导入
if [ ! -n "$1" ] ;then
  echo "请输入次数!! 如: ./postgresql-insert-data.sh 2000"
else
while [ $i -le $MAX_INSERT_ROW_COUNT ]
do
     server_name="$db_name$i"
     machine_code=1000$i
     echo "$server_name:$machine_code"
     i=$(($i+1))
     sleep 0.2
     psql -U postgres -d $db_name -h 127.0.0.1 -c "INSERT INTO $table_name(id, server_name,auth_code) VALUES ('$i','$server_name','$machine_code')"
done
exit 0
fi
```

---

* 批量插入数据

```
postgres@postgresql-0:/tmp$ sh postgresql-insert-data.sh 10000
authdb1:10001
INSERT 0 1
authdb2:10002
INSERT 0 1
authdb3:10003
INSERT 0 1
authdb4:10004
INSERT 0 1
authdb5:10005
INSERT 0 1
authdb6:10006
INSERT 0 1
authdb7:10007
INSERT 0 1
authdb8:10008
INSERT 0 1
authdb9:10009
INSERT 0 1
authdb10:100010
INSERT 0 1
authdb11:100011
INSERT 0 1
authdb12:100012
INSERT 0 1
authdb13:100013
INSERT 0 1
authdb14:100014
INSERT 0 1
authdb15:100015
INSERT 0 1
authdb16:100016
INSERT 0 1
authdb17:100017
INSERT 0 1
authdb18:100018

```
---

> 删除从库测试


* 操作从库

```
root@shell:/srv/ops-kubeconfig# kubectl delete pods postgresql-1 -n pgsql
pod "postgresql-1" deleted



root@shell:/srv/ops-kubeconfig# kubectl get pods -n pgsql -w
NAME           READY   STATUS    RESTARTS   AGE
postgresql-0   2/2     Running   0          51m
postgresql-1   0/2     Init:1/2   0          9s
postgresql-2   2/2     Running   0          49m
postgresql-1   0/2     Running   0          11s
postgresql-1   1/2     Running   0          41s
postgresql-1   2/2     Running   0          48s

```

* 查看同步情况

```
postgres@postgresql-0:/$ psql 
psql (9.6.16)
Type "help" for help.

postgres=# select  pid ,usesysid,usename,client_addr,state,sync_state  from  pg_stat_replication;
 pid  | usesysid | usename  |   client_addr   |   state   | sync_state 
------+----------+----------+-----------------+-----------+------------
 8896 |       10 | postgres | 172.110.101.125 | streaming | async
   81 |       10 | postgres | 172.110.101.123 | streaming | async
(2 rows)
```


* 查看从库数据

```
postgres@postgresql-1:/$ psql 
psql (9.6.16)
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 authdb    | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)

postgres=# \c authdb;
You are now connected to database "authdb" as user "postgres".

authdb=# select * from server_auth_info;
 id  |                    server_name                     |                     auth_code                      
-----+----------------------------------------------------+----------------------------------------------------
   2 | authdb1                                            | 10001                                             
   3 | authdb2                                            | 10002                                             
   4 | authdb3                                            | 10003
  ...........................
  99 | authdb98                                           | 100098                                            

```


---

> 删除主库测试


* 操作主库


```
root@shell:/srv/ops-kubeconfig# kubectl delete pods postgresql-0 -n pgsql
pod "postgresql-0" deleted


root@shell:/srv/ops-kubeconfig# kubectl get pods -n pgsql
NAME           READY   STATUS    RESTARTS   AGE
postgresql-0   2/2     Running   0          2m5s
postgresql-1   2/2     Running   0          6m45s
postgresql-2   2/2     Running   0          56m

```

---

* 查看主从同步情况


```
postgres@postgresql-0:/$ psql 
psql (9.6.16)
Type "help" for help.

postgres=# select  pid ,usesysid,usename,client_addr,state,sync_state  from  pg_stat_replication;
 pid | usesysid | usename  |   client_addr   |   state   | sync_state 
-----+----------+----------+-----------------+-----------+------------
  29 |       10 | postgres | 172.110.101.123 | streaming | async
  30 |       10 | postgres | 172.110.101.125 | streaming | async
(2 rows)

```


---

* 往主从插入两条数据


```
postgres@postgresql-0:/$ psql 
psql (9.6.16)
Type "help" for help.

postgres=# insert into test2 values(3,'aaa'),(4,'abc');
INSERT 0 2
postgres=# insert into test2 values(5,'aaa'),(6,'abc');
INSERT 0 2
postgres=# select * from test2;
 id | name 
----+------
  1 | aaa
  2 | abc
  3 | aaa
  4 | abc
  5 | aaa
  6 | abc
(6 rows)

```

* 查看从库的数据

```
postgres@postgresql-1:/$ psql 
psql (9.6.16)
Type "help" for help.

postgres=# select * from test2;
 id | name 
----+------
  1 | aaa
  2 | abc
  3 | aaa
  4 | abc
  5 | aaa
  6 | abc
(6 rows)

```
