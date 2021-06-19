+++
author = "Ethon Wu"
title = "PostgreSQL replication 機制設定 (一)"
date = "2020-11-17T13:48:00+08:00"
description = "PostgreSQL replication 機制設定 (一)"
tags = [
    "PostgreSQL","streaming replication"
]
+++



# PostgreSQL replication 機制設定

* 最近研究了 PostgreSQL replication 的機制設定，因此寫了這篇文章，將當時的架設過程記錄下來。
* PostgreSQL streaming replication 簡單來說，就是轉被兩個 DB ，一個 DB 為 Master （主DB），主要負責所有操作，另一個 DB 為 Slave (從DB)，它的功能只有負責接收 Master （主DB）傳送過來的資料，本身這個 DB 只能 select 其他的功能都沒有，往後的 pgpool 、patroni 架構，都是基於 **streaming replication** 架構去往後延伸。

# 設定過程

* 首先我們會準備兩個節點，並且將 PostgreSQL 架設在上面，並且設定 streaming replication 的抄寫

## 環境介紹

* OS: CentOS 7.8
* Host * 2


| Hostname  | IP             | Description                                      |
| --------- | -------------- | ------------------------------------------------ |
| ethon.mh01| 10.37.0.2      | PostgreSQL Master DB			        |
| ethon.mh02| 10.37.0.3      | PostgreSQL Slave  DB				|

* PostgreSQL rpm version ( 安裝方法是使用 rpm 的方式進行安裝 repo 來源是使用原生 DVD 底下的套件 )， rpm 安裝的 postgres home 目錄在 /var/lib/pgsql 底下
    

    [root@ethon ~]# rpm -qa | grep postgresql
    postgresql-libs-9.2.24-4.el7_8.x86_64
    postgresql-server-9.2.24-4.el7_8.x86_64
    postgresql-contrib-9.2.24-4.el7_8.x86_64
    postgresql-9.2.24-4.el7_8.x86_64

### Master 設定

#### 1. 安裝 PostgreSQL ，並且設定所需語系，可以按照需求去更改語系

    
    [root@ethonwu ~]# yum install -y postgresql-server
    [root@ethonwu ~]# export LANGUAGE=en_US.UTF-8
    [root@ethonwu ~]# export LANG=en_US.UTF-8
    [root@ethonwu ~]# export LC_ALL=en_US.UTF-8
    [root@ethonwu ~]# echo 'LC_ALL="en_US.UTF-8"' >> /etc/locale.conf
    [root@ethonwu ~]# sudo su -l postgres -c "postgresql-setup initdb"
    Initializing database ... OK
    [root@ethonwu ~]# systemctl start postgresql


#### 2. 創建 Replication 角色

* 設定 streaming replication 過程中，需要創建一個 **角色 (role)** 來處理抄寫資料到 Slave 的大小事，在 PostgreSQL 中，除了需要建立此角色以外，也需要在 pg_hba 設定檔設定規則
* root 不能直接登入至 PostgreSQL 中，必須切換到 postgres 身份才可以使用 admin 身份訪問 DB

建立 replica 的角色，密碼簡單設定為 1qaz2wsx 
   
   
    [root@ethonwu ~]# sudo -u postgres psql
    could not change directory to "/root"
    psql (9.2.24)
    Type "help" for help.
    
    postgres=# CREATE ROLE replica login replication encrypted password '1qaz2wsx';
    CREATE ROLE
    postgres=# \du
                                 List of roles
     Role name |                   Attributes                   | Member of
    -----------+------------------------------------------------+-----------
     postgres  | Superuser, Create role, Create DB, Replication | {}
     replica   | Replication                                    | {}


進入 pg_hba.conf 設定規則，規則需要設定 Slave (10.37.0.3) 的 IP ，允許透過這個 IP 訪問到 Master ，並進行資料抄寫


    [root@ethonwu ~]# su - postgres
    -bash-4.2$ cat data/pg_hba.conf | grep -v "#\|^$"
    local   all             all                                     peer
    host    all             all             127.0.0.1/32            ident
    host    all             all             ::1/128                 ident
    host    replication     replica     10.37.0.3/32                 md5


#### 3. 更改 PostgreSQL 設定檔

進入 postgresql.conf 設定檔中，更改以下欄位得值

    
    listen_addresses = '*'
    wal_level = hot_standby
    max_wal_senders = 32
    wal_keep_segments = 256

設定命令，將剛剛上述的三個參數設定到 data/postgresql.conf 中，並且重啟服務

    
    [root@ethonwu ~]# su - postgres
    Last login: Mon Jun 14 09:45:49 UTC 2021 on pts/0
    -bash-4.2$ vim data/postgresql.conf
    -bash-4.2$ logout
    [root@ethonwu ~]# systemctl restart postgresql


使用 ps aux 查看 PostgreSQL 相關 Process 有沒有變化，多了 streaming replication 相關的 Process

    
    [root@ethonwu ~]# ps aux | grep postgres
    postgres  9999  0.0  0.1 134920  9516 ?        S    10:05   0:00 /usr/bin/postgres -D /var/lib/pgsql/data -p 5432
    postgres 10000  0.0  0.0  94668  1524 ?        Ss   10:05   0:00 postgres: logger process
    postgres 10002  0.0  0.0 134920  1728 ?        Ss   10:05   0:00 postgres: checkpointer process
    postgres 10003  0.0  0.0 134920  1732 ?        Ss   10:05   0:00 postgres: writer process
    postgres 10004  0.0  0.0 134920  1496 ?        Ss   10:05   0:00 postgres: wal writer process
    postgres 10005  0.0  0.0 135752  2876 ?        Ss   10:05   0:00 postgres: autovacuum launcher process
    postgres 10006  0.0  0.0  94796  1660 ?        Ss   10:05   0:00 postgres: stats collector process
    root     10072  0.0  0.0  12532   972 pts/0    S+   10:05   0:00 grep --color=auto postgres

 
### Slave 設定

#### 1. 這邊依然安裝 DB ， 但是不對他做 init DB 的動作


    [root@localhost ~]# yum install -y postgresql-server
    [root@localhost ~]# export LANGUAGE=en_US.UTF-8
    [root@localhost ~]# export LANG=en_US.UTF-8
    [root@localhost ~]# export LC_ALL=en_US.UTF-8
    [root@localhost ~]# echo 'LC_ALL="en_US.UTF-8"' >> /etc/locale.conf

#### 2. 複製 Master 資料到 Slave 上

* 進入 postgres 的 home 目錄，使用 pg_basebackup 命令，透過在 Master 上創建的角色 replica 將資料複製到 Slave 的 postgres home 目錄


``` 
[root@localhost ~]# su - postgres
Last login: Mon Jun 14 11:01:30 UTC 2021 on pts/0
-bash-4.2$ cd data/
-bash-4.2$ ls -al
total 0
drwx------ 2 postgres postgres  6 May  6 15:00 .
drwx------ 4 postgres postgres 75 Jun 14 10:59 ..
-bash-4.2$ pwd
/var/lib/pgsql/data
-bash-4.2$ pg_basebackup -F p -Xs --progress -D /var/lib/pgsql/data -h 10.37.0.2 -p 5432 -U replica --password
Password:
19362/19362 kB (100%), 1/1 tablespace
-bash-4.2$ ls
backup_label  global   pg_hba.conf    pg_log        pg_notify  pg_snapshots  pg_subtrans  pg_twophase  pg_xlog
base          pg_clog  pg_ident.conf  pg_multixact  pg_serial  pg_stat_tmp   pg_tblspc    PG_VERSION   postgresql.conf
```


---


#### 3. 修改 Slave 相關設定檔 

* pg_basebackup 命令會將 Master postgres 的 home 目錄底下的所有資料搬過來， Slave 設定檔會需要額外修改，這邊就先備份 Master 備份過來得資料，再進行更改
  
---
  
備份設定檔

    
    -bash-4.2$ ls
    backup_label  global   pg_hba.conf    pg_log        pg_notify  pg_snapshots  pg_subtrans  pg_twophase  pg_xlog
    base          pg_clog  pg_ident.conf  pg_multixact  pg_serial  pg_stat_tmp   pg_tblspc    PG_VERSION   postgresql.conf
    -bash-4.2$ cd ../
    -bash-4.2$ pwd
    /var/lib/pgsql
    -bash-4.2$ mkdir back_master_config
    -bash-4.2$ cp -p data/postgresql.conf data/pg_hba.conf back_master_config/
    -bash-4.2$  ls -al back_master_config/
    total 28
    drwxr-xr-x 2 postgres postgres    48 Jun 14 13:32 .
    drwx------ 5 postgres postgres   101 Jun 14 13:32 ..
    -rw------- 1 postgres postgres  4301 Jun 14 11:03 pg_hba.conf
    -rw------- 1 postgres postgres 19800 Jun 14 11:03 postgresql.conf
    -bash-4.2$


更改 postgresql.conf 設定檔和創建 recovery.conf

postgresql.conf 更改內容
max_connections 的部分要大於 Master 的 max_connections 的配置
    
    
    max_connections = 2000
    hot_standby = on
    max_standby_streaming_delay = 30s
    wal_receiver_status_interval = 1s
    hot_standby_feedback = on


recovery.conf 更改內容
primary_conninfo 輸入 Master 的 host, port,創建的 role 名稱和密碼
    
    
    standby_mode = on
    primary_conninfo = 'host=10.37.0.2 port=5432 user=replica password=<password>'
    recovery_target_timeline = 'latest'


編輯設定檔命令，並且重啟服務

    
    -bash-4.2$ vim data/postgresql.conf
    -bash-4.2$ vim data/recovery.conf
    -bash-4.2$ pwd
    /var/lib/pgsql
    -bash-4.2$ logout
    [root@localhost ~]# systemctl restart postgresql
    [root@localhost ~]#


### 查看是否有完成 streaming replication

* 從 Process 查看

#### Master 部分 


    [root@ethonwu ~]# hostname
    ethonwu.mh01
    [root@ethonwu ~]# ps aux | grep postgres
    postgres 16785  0.0  0.0 135760  3160 ?        Ss   13:40   0:00 postgres: wal sender process replica 10.37.0.3(40182) streaming 0/30016B0
    postgres 20252  0.0  0.1 134920  9528 ?        S    11:01   0:00 /usr/bin/postgres -D /var/lib/pgsql/data -p 5432
    postgres 20253  0.0  0.0  92544  1512 ?        Ss   11:01   0:00 postgres: logger process
    postgres 20255  0.0  0.0 134920  2248 ?        Ss   11:01   0:00 postgres: checkpointer process
    postgres 20256  0.0  0.0 134920  2000 ?        Ss   11:01   0:00 postgres: writer process
    postgres 20257  0.0  0.0 134920  1504 ?        Ss   11:01   0:00 postgres: wal writer process
    postgres 20258  0.0  0.0 135776  2928 ?        Ss   11:01   0:00 postgres: autovacuum launcher process
    postgres 20259  0.0  0.0  94796  1732 ?        Ss   11:01   0:00 postgres: stats collector process
    root     22936  0.0  0.0  12504   952 pts/0    S+   14:15   0:00 grep --color=auto postgres


#### Slave 部分 


    [root@ethonwu ~]# hostname
    ethonwu.mh02
    [root@ethonwu ~]# ps aux | grep postgres
    postgres 14689  0.0  0.8 320332 65692 ?        S    13:42   0:00 /usr/bin/postgres -D /var/lib/pgsql/data -p 5432
    postgres 14690  0.0  0.0 192828  1508 ?        Ss   13:42   0:00 postgres: logger process
    postgres 14691  0.0  0.0 320380  2056 ?        Ss   13:42   0:00 postgres: startup process   recovering 000000010000000000000003
    postgres 14692  0.0  0.0 320332  1988 ?        Ss   13:42   0:00 postgres: checkpointer process
    postgres 14693  0.0  0.0 320332  1992 ?        Ss   13:42   0:00 postgres: writer process
    postgres 14694  0.0  0.0 194948  1572 ?        Ss   13:42   0:00 postgres: stats collector process
    postgres 14695  0.0  0.0 327060  3068 ?        Ss   13:42   0:01 postgres: wal receiver process   streaming 0/30016B0
    root     14787  0.0  0.0 112808   976 pts/0    S+   14:18   0:00 grep --color=auto postgres


---

* 進入 DB 查看

#### 進入 Master 查看

    [root@ethonwu ~]# hostname
    ethonwu.mh01
    [root@ethonwu ~]# sudo -u postgres psql
    could not change directory to "/root"
    psql (9.2.24)
    Type "help" for help.
    
    postgres=# select * from pg_stat_replication;
      pid  | usesysid | usename | application_name | client_addr | client_hostname | client_port |         backend_start         |   state   | sent_location | write_location | flush_location | replay_location
     | sync_priority | sync_state
    -------+----------+---------+------------------+-------------+-----------------+-------------+-------------------------------+-----------+---------------+----------------+----------------+----------------
    -+---------------+------------
     16785 |    16384 | replica | walreceiver      | 10.37.0.3   |                 |       40182 | 2021-06-14 13:40:53.046265+00 | streaming | 0/3001748     | 0/3001748      | 0/3001748      | 0/3001748
     |             0 | async
    (1 row)
    
    postgres=#



完成了 streaming replication 的設定後，下一篇文章，將介紹如何在 streaming replication 模式下，手動切換主從
