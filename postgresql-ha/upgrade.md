# PostgreSQL  9.6 to 12.4  Data Upgrade

## 初始化新版本的数据文件

* 仅初始化数据, 不启动服务

```
docker run --rm -e POSTGRES_PASSWORD=postgresql -e PGDATA=/var/lib/postgresql/data/pgdata -v /home/data/:/var/lib/postgresql/data postgres:12.4 - docker_init_database_dir


```


---

* 数据文件


```
总用量 124
drwx------ 19 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 ./
drwxrwxr-x  3 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 ../
drwx------  5 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 base/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 global/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_commit_ts/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_dynshmem/
-rw-------  1 zhihuanzhu zhihuanzhu  4782 9月  21 09:04 pg_hba.conf
-rw-------  1 zhihuanzhu zhihuanzhu  1636 9月  21 09:04 pg_ident.conf
drwx------  4 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_logical/
drwx------  4 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_multixact/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_notify/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_replslot/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_serial/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_snapshots/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_stat/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_stat_tmp/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_subtrans/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_tblspc/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_twophase/
-rw-------  1 zhihuanzhu zhihuanzhu     3 9月  21 09:04 PG_VERSION
drwx------  3 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_wal/
drwx------  2 zhihuanzhu zhihuanzhu  4096 9月  21 09:04 pg_xact/
-rw-------  1 zhihuanzhu zhihuanzhu    88 9月  21 09:04 postgresql.auto.conf
-rw-------  1 zhihuanzhu zhihuanzhu 26588 9月  21 09:04 postgresql.conf

```

---

## 备份旧版本数据库文件


* 需要先停止数据库, 直接拷贝 data 目录文件, 在 upgrade 的时候会报错.


  * `pg_ctl -D ../data/ stop -s -m fast`


  * 执行 `pg_controldata` 查看 数据库是否关闭


  * 拷贝数据目录到 与新版本的数据库同一个目录中.

---

* 由于 postgresql 未关闭导致的报错如下:


  * `The source cluster was not shut down cleanly.`


  * `There seems to be a postmaster servicing the old cluster. Please shutdown` 

---

```
# postgresql 新旧版本数据目录结构

/mnt/postgresql/{9.6/data, 12/data}

```

---

## 执行 Upgrade 程序

* upgrade 需要如下资源
  
  * 旧版本程序 既 postgresql-9.6/bin 目录下的文件.

  * 新版本程序 既 postgresql-12/bin 目录下的文件.

  * 旧版本数据文件 既 data 目录文件.

  * 新版本初始化数据文件 既 initdb 后的 data 目录文件. 

---

* 这里使用一个 docker 的镜像用于 upgrade, 这样不需要破坏原环境, 也可以同时包含新旧版本的程序.


  * github 地址: https://github.com/tianon/docker-postgres-upgrade


  * 选择对应版本的 镜像.


  * 注意事项:


    * 旧版本中的 postgresql.conf 配置文件中的  `lc_monetary`、`lc_numeric`、`lc_time` 请修改为 'en_US.utf8'.


```
docker run --rm \
  -v /mnt/postgresql:/var/lib/postgresql \
  tianon/postgres-upgrade:9.6-to-12 \
  --link
```

---


* 输出如下表示完成:

```
Setting next OID for new cluster                            ok
Sync data directory to disk                                 ok
Creating script to analyze new cluster                      ok
Creating script to delete old cluster                       ok
Checking for hash indexes                                   ok

Upgrade Complete
----------------
Optimizer statistics are not transferred by pg_upgrade so,
once you start the new server, consider running:
    ./analyze_new_cluster.sh

Running this script will delete the old cluster's data files:
    ./delete_old_cluster.sh
```

---

## 拷贝旧版本的配置文件到新版本中

* 配置文件 (大版本中配置文件有变化, 可对照旧版本的配置文件修改到新版本的配置文件中)

  * `postgresql.conf`

  * `pg_hba.conf` 

---


## 启动新版本


* 启动程序

  * 修改目录 cp -r /mnt/postgresql/12/data /mnt/postgresql/data/pgdata

  * `docker run -d --name postgresql -e PGDATA=/var/lib/postgresql/data/pgdata -v /mnt/postgresql/data:/var/lib/postgresql/data postgres:12.4`

---

* 更新收集统计信息, pg_upgrade 不会将 统计信息 迁移过来, 需要重新统计

  * 进入容器 `docker exec -it postgresql bash`

  * 切换用户 su - postgres

  * 执行命令  `vacuumdb --all --analyze-in-stages -h localhost -p 5432`

  *  检查新版本 posgresql 数据

  * 导出数据库为 sql 文件  `pg_dumpall > postgresql-all-12.sql`

  * 还原到 postgresql 12 版本

---

* 如下为 vacuumdb 操作后的输出

```
postgres@c55276f864b7:~$ vacuumdb --all --analyze-in-stages -h localhost -p 5432
vacuumdb: processing database "postgres": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "template1": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "test": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "postgres": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "template1": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "test": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "postgres": Generating default (full) optimizer statistics
vacuumdb: processing database "template1": Generating default (full) optimizer statistics
vacuumdb: processing database "test": Generating default (full) optimizer statistics

```
