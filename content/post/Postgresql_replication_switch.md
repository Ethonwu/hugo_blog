+++
author = "Ethon Wu"
title = "PostgreSQL replication 機制切換"
date = "2020-11-17T13:48:00+08:00"
description = "PostgreSQL replication 機制切換"
tags = [
    "PostgreSQL","streaming replication"
]
+++



# PostgreSQL replication 機制設定

* 根據上一篇 [PostgreSQL replication 的機制設定](https://ethonwu.github.io/post/postgresql_server_replication/)，有提到如何設定以及架設基本的 streaming replication 功能，這邊來介紹 **如何將 Slave 節點切換成 Master 節點** 


# 切換過程

1. 首先會在 Master ( ethon.mh01 ) 建立一個 DB ，將 Master DB ( ethon.mh01 ) 關閉，將 Slave ( ethon.mh02 ) 節點設定成 Master ( ethon.mh02 ) ，寫入資料到 Master ( ethon.mh02 ) 上
2. 在將原本的 Master ( ethon.mh02 ) 切換回 Slave 

## 環境介紹

* OS: CentOS 7.8
* Host * 2


| Hostname  | IP             | Description                                      |
| --------- | -------------- | ------------------------------------------------ |
| ethon.mh01| 10.37.0.2      | PostgreSQL Master DB			        |
| ethon.mh02| 10.37.0.3      | PostgreSQL Slave  DB				|

* PostgreSQL rpm version ( 安裝方法是使用 rpm 的方式進行安裝 repo 來源是使用原生 DVD 底下的套件 )， rpm 安裝的 postgres home 目錄在 /var/lib/pgsql 底下
    
```
[root@ethon ~]# rpm -qa | grep postgresql
postgresql-libs-9.2.24-4.el7_8.x86_64
postgresql-server-9.2.24-4.el7_8.x86_64
postgresql-contrib-9.2.24-4.el7_8.x86_64
postgresql-9.2.24-4.el7_8.x86_64
```

### 在 Master 上建立 DB ，並且關閉服務

* 將 Master 關閉，並且將 Slave ( ethon.mh02 ) 機器上的 PostgreSQL 切換成 Master，**再 Slave 切換成 Master 之前該機器上的 Postgresql 都是 read-only 的模式，無法寫入資料**

#### 建立 master_db 名稱的 DB ，並且關閉 Master ( ethonwu.mh01 ) 服務

    
    [root@ethonwu ~]# hostname
    ethonwu.mh01
    [root@ethonwu ~]# su - postgres
    -bash-4.2$ psql
    psql (9.2.24)
    Type "help" for help.
    
    postgres=# create database master_db;
    CREATE DATABASE
    postgres=# \l
                                 List of databases
       Name    |  Owner   | Encoding  | Collate | Ctype |   Access privileges
    -----------+----------+-----------+---------+-------+-----------------------
     master_db | postgres | SQL_ASCII | C       | C     |
     postgres  | postgres | SQL_ASCII | C       | C     |
     template0 | postgres | SQL_ASCII | C       | C     | =c/postgres          +
               |          |           |         |       | postgres=CTc/postgres
     template1 | postgres | SQL_ASCII | C       | C     | =c/postgres          +
               |          |           |         |       | postgres=CTc/postgres
    (4 rows)
    postgres=# \q
    -bash-4.2$ logout
    [root@ethonwu ~]#systemctl stop postgresql
    [root@ethonwu ~]# systemctl status postgresql
    ● postgresql.service - PostgreSQL database server
       Loaded: loaded (/usr/lib/systemd/system/postgresql.service; disabled; vendor preset: disabled)
       Active: inactive (dead)
    
    Jun 14 10:05:26 ethonwu.mh01 systemd[1]: Stopping PostgreSQL database server...
    Jun 14 10:05:27 ethonwu.mh01 systemd[1]: Stopped PostgreSQL database server.
    Jun 14 10:05:27 ethonwu.mh01 systemd[1]: Starting PostgreSQL database server...
    Jun 14 10:05:28 ethonwu.mh01 systemd[1]: Started PostgreSQL database server.
    Jun 14 11:01:13 ethonwu.mh01 systemd[1]: Stopping PostgreSQL database server...
    Jun 14 11:01:14 ethonwu.mh01 systemd[1]: Stopped PostgreSQL database server.
    Jun 14 11:01:14 ethonwu.mh01 systemd[1]: Starting PostgreSQL database server...
    Jun 14 11:01:15 ethonwu.mh01 systemd[1]: Started PostgreSQL database server.
    Jun 19 13:38:54 ethonwu.mh01 systemd[1]: Stopping PostgreSQL database server...
    Jun 19 13:38:55 ethonwu.mh01 systemd[1]: Stopped PostgreSQL database server.
    [root@ethonwu ~]#
    


#### 更改設定檔，讓 Slave ( ethon.mh02 ) 切換成 Master ( ethon.mh01 ) 

* 將 Slave 的設定檔進行備份，將一開始的 Master 設定檔與現有的設定檔進行交換，這邊要將 **recovery.conf** 拿掉
* back_master_config 這個資料夾是在上一章節，從 Master 將資料複製過來所建立的資料夾，如果沒有看上一篇文章，這邊可能不會有建立資料夾的紀錄
    
```    
[root@ethonwu ~]# hostname
ethonwu.mh02
[root@ethonwu ~]# su - postgres
-bash-4.2$ pwd
/var/lib/pgsql
-bash-4.2$ mkdir back_slave_config
-bash-4.2$ cp -p data/postgresql.conf data/pg_hba.conf back_slave_config/
-bash-4.2$ mv data/recovery.conf back_slave_config/
-bash-4.2$ ls
back_master_config  back_slave_config  backups  data
# #  將一開始的 Master 設定檔與現有的設定檔進行交換
-bash-4.2$ cp -p back_master_config/postgresql.conf back_master_config/pg_hba.conf data/
```

* 設定 Slave 上面的 pg_hba.conf 檔案，因為是從原本的 Master 複製過來，因此 pg_hba 需要更改

將 10.37.0.3 改成 10.37.0.2

    
    -bash-4.2$ vim data/pg_hba.conf
    -bash-4.2$ cat data/pg_hba.conf | grep -v "#\|^$"
    local   all             all                                     peer
    host    all             all             127.0.0.1/32            ident
    host    all             all             ::1/128                 ident
    host    replication     replica     10.37.0.2/32                 md5
    

* 啟動 ethonwu.mh02 上的 PostgreSQL 


```    
-bash-4.2$ logout
[root@ethonwu ~]# systemctl restart postgresql
[root@ethonwu ~]# su - postgres
-bash-4.2$ psql
psql (9.2.24)
Type "help" for help.

postgres=# \l
                             List of databases
   Name    |  Owner   | Encoding  | Collate | Ctype |   Access privileges
-----------+----------+-----------+---------+-------+-----------------------
 master_db | postgres | SQL_ASCII | C       | C     |
 postgres  | postgres | SQL_ASCII | C       | C     |
 template0 | postgres | SQL_ASCII | C       | C     | =c/postgres          +
           |          |           |         |       | postgres=CTc/postgres
 template1 | postgres | SQL_ASCII | C       | C     | =c/postgres          +
           |          |           |         |       | postgres=CTc/postgres
(4 rows)

postgres=#
```




* 並且針對剛才設定的 DB  建立 Table 以及寫入資料


``` 
postgres=# \c master_db
You are now connected to database "master_db" as user "postgres".
master_db=# CREATE TABLE switch_to_master ( NAME TEXT NOT NULL );
CREATE TABLE
master_db=# \d
              List of relations
 Schema |       Name       | Type  |  Owner
--------+------------------+-------+----------
 public | switch_to_master | table | postgres
(1 row)   
master_db=# INSERT INTO switch_to_master (name) VALUES ('ethon');
INSERT 0 1
master_db=# select * from switch_to_master;
 name
-------
 ethon
(1 row)

```


* 上述切換過程，成功將 Slave 切換成 Master ，有了 insert 的功能，並不能只有 select 的功能

### 把服務改回原本狀況

* 目前 Master ethonwu.mh02 ，ethonwu.mh01 服務未啟動，改回 Master ( ethonwu.mh01 ) 、Slave ( ethonwu.mh02 ) 模式 

#### 登入 ethonwu.mh01 進行設定

* 進入 Master ( ethonwu.mh01 ) 更改 data 目錄名稱，並且從新新建 data 資料夾設定權限為 700，因為在做 pg_basebackup 的時候，目錄必須為空

    
```
[root@ethonwu ~]# su - postgres
-bash-4.2$ pwd
/var/lib/pgsql
-bash-4.2$ mv data/ data_backup/
-bash-4.2$ mkdir data
-bash-4.2$ ll data
total 0
-bash-4.2$ ls -al
total 24
drwx------   5 postgres postgres  149 Jun 19 14:18 .
drwxr-xr-x. 32 root     root     4096 Jun 14 09:32 ..
-rw-------   1 postgres postgres  144 Jun 19 13:38 .bash_history
-rw-r--r--   1 postgres postgres   85 May  6 15:00 .bash_profile
-rw-------   1 postgres postgres  205 Jun 19 13:38 .psql_history
-rw-------   1 postgres postgres 1185 Jun 14 11:00 .viminfo
drwx------   2 postgres postgres    6 May  6 15:00 backups
drwxr-xr-x   2 postgres postgres    6 Jun 19 14:18 data
drwx------  15 postgres postgres  328 Jun 19 13:38 data_backup
-rw-------   1 postgres postgres 1223 Jun 14 09:32 initdb.log
-bash-4.2$ chmod 700 data/
```


* 將 ethonwu.mh02 上的資料備份回來


```    
[root@ethonwu ~]# su - postgres
-bash-4.2$ cd data
-bash-4.2$ ls
-bash-4.2$ pg_basebackup -F p -Xs --progress -D /var/lib/pgsql/data -h 10.37.0.3 -p 5432 -U replica --password
Password:
25796/25796 kB (100%), 1/1 tablespace
-bash-4.2$ ls
PG_VERSION    backup_label.old  global   pg_hba.conf    pg_log        pg_notify  pg_snapshots  pg_subtrans  pg_twophase  postgresql.conf
backup_label  base              pg_clog  pg_ident.conf  pg_multixact  pg_serial  pg_stat_tmp   pg_tblspc    pg_xlog
```


* 同上更改 ethonwu.mh02 上 pg_hba.conf 設定檔


```    
[root@ethonwu ~]# hostname
ethonwu.mh01
[root@ethonwu ~]# su - postgres
-bash-4.2$ vim data/pg_hba.conf
-bash-4.2$ cat data/pg_hba.conf | grep -v "#\|^$"
local   all             all                                     peer
host    all             all             127.0.0.1/32            ident
host    all             all             ::1/128                 ident
host    replication     replica     10.37.0.3/32                 md5
-bash-4.2$ logout
```


#### 將 ethonwu.mh02 改回 Slave 模式

* 進入 ethonwu.mh02  將目前的 Master 模式切換成 Slave 模式，讓他去認 Master ( ethonwu.mh01 ) 的資料，並且關閉服務


```    
[root@ethonwu ~]# su - postgres
-bash-4.2$ cp -p back_slave_config/*.conf data/
-bash-4.2$ ls -al data/ | grep conf
-rw-------  1 postgres postgres  4301 Jun 14 11:03 pg_hba.conf
-rw-------  1 postgres postgres  1636 Jun 14 11:03 pg_ident.conf
-rw-------  1 postgres postgres 19812 Jun 14 13:41 postgresql.conf
-rw-r--r--  1 postgres postgres   131 Jun 14 13:42 recovery.conf
-bash-4.2$ logout
[root@ethonwu ~]# systemctl stop postgresql
```


### 分別啟動 ethonwu.mh01 和 ethonwu.mh02 上的 PostgreSQL 

* 啟動 PostgreSQL 並且查看剛剛在 ethonwu.mh02 上新增的資料是否完整
啟動 ethonwu.mh01 上的 PostgreSQL


```    
[root@ethonwu ~]# hostname
ethonwu.mh01
[root@ethonwu ~]# systemctl restart postgresql
[root@ethonwu ~]# systemctl status postgresql
● postgresql.service - PostgreSQL database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2021-06-19 14:34:10 UTC; 4s ago
  Process: 1443 ExecStart=/usr/bin/pg_ctl start -D ${PGDATA} -s -o -p ${PGPORT} -w -t 300 (code=exited, status=0/SUCCESS)
  Process: 1438 ExecStartPre=/usr/bin/postgresql-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 1447 (postgres)
    Tasks: 7
   Memory: 25.0M
   CGroup: /system.slice/postgresql.service
           ├─1447 /usr/bin/postgres -D /var/lib/pgsql/data -p 5432
           ├─1448 postgres: logger process
           ├─1450 postgres: checkpointer process
           ├─1451 postgres: writer process
           ├─1452 postgres: wal writer process
           ├─1453 postgres: autovacuum launcher process
           └─1454 postgres: stats collector process

Jun 19 14:34:09 ethonwu.mh01 systemd[1]: Starting PostgreSQL database server...
Jun 19 14:34:10 ethonwu.mh01 systemd[1]: Started PostgreSQL database server.
```


啟動 ethonwu.mh02 上的 PostgreSQL



    
    [root@ethonwu ~]# hostname
    ethonwu.mh02
    [root@ethonwu ~]# systemctl start postgresql
    [root@ethonwu ~]# systemctl status postgresql
    ● postgresql.service - PostgreSQL database server
       Loaded: loaded (/usr/lib/systemd/system/postgresql.service; disabled; vendor preset: disabled)
       Active: active (running) since 六 2021-06-19 14:39:02 UTC; 6s ago
      Process: 19424 ExecStart=/usr/bin/pg_ctl start -D ${PGDATA} -s -o -p ${PGPORT} -w -t 300 (code=exited, status=0/SUCCESS)
      Process: 19419 ExecStartPre=/usr/bin/postgresql-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
     Main PID: 19427 (postgres)
        Tasks: 7
       Memory: 62.4M
       CGroup: /system.slice/postgresql.service
               ├─19427 /usr/bin/postgres -D /var/lib/pgsql/data -p 5432
               ├─19428 postgres: logger process
               ├─19429 postgres: startup process   waiting for 000000010000000000000005
               ├─19430 postgres: checkpointer process
               ├─19431 postgres: writer process
               ├─19432 postgres: stats collector process
               └─19433 postgres: wal receiver process
    
     6月 19 14:39:01 ethonwu.mh02 systemd[1]: Starting PostgreSQL database server...
     6月 19 14:39:02 ethonwu.mh02 systemd[1]: Started PostgreSQL database server.



### 查看 Master ( ethonwu.mh01 ) 上的 Replica 狀態和 Table 資料


* 檢查 Table 資料


```    
[root@ethonwu ~]# su - postgres
-bash-4.2$ psql
psql (9.2.24)
Type "help" for help.

postgres=# \l
                             List of databases
   Name    |  Owner   | Encoding  | Collate | Ctype |   Access privileges
-----------+----------+-----------+---------+-------+-----------------------
 master_db | postgres | SQL_ASCII | C       | C     |
 postgres  | postgres | SQL_ASCII | C       | C     |
 template0 | postgres | SQL_ASCII | C       | C     | =c/postgres          +
           |          |           |         |       | postgres=CTc/postgres
 template1 | postgres | SQL_ASCII | C       | C     | =c/postgres          +
           |          |           |         |       | postgres=CTc/postgres
(4 rows)

postgres=# \c master_db
You are now connected to database "master_db" as user "postgres".
master_db=# select * from switch_to_master;
 name
-------
 ethon
(1 row)
```


* 查看 Replica 狀態


```    
master_db=# select * from pg_stat_replication;
pid  | usesysid | usename | application_name | client_addr | client_hostname | client_port |         backend_start         |   state   | sent_location | write_location | flush_location | replay_location
| sync_priority | sync_state
------+----------+---------+------------------+-------------+-----------------+-------------+-------------------------------+-----------+---------------+----------------+----------------+-----------------
+---------------+------------
 1941 |    16384 | replica | walreceiver      | 10.37.0.3   |                 |       40814 | 2021-06-19 14:36:55.704139+00 | streaming | 0/5000080     | 0/5000080      | 0/5000118      | 0/5000118
|             0 | async
(1 row)
``` 



成功完成所有的切換，以及測試，切換過一次會有一種想法：我是否可以寫一隻腳本，來搬移 Master 和 Slave 上的設定檔，來達到自動切換呢？ 以及延伸出 **該如何做一個 Proxy 來讓 client 不需要指定 IP 就可以直接訪問到 Master 並且寫入資料** ，因此下一章節會介紹 **pgpool-II** 的架構，這個架構將上述的兩個問題解決了 自動切換腳本 和 Proxy 。
