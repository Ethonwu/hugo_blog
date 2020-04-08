+++
author = "Ethon Wu"
title = "Hive Sentry Setting"
date = "2020-03-16T18:00:22+08:00"
description = "Enable MIT Kerberos 後，介紹如何整合 Sentry 透過 Hive Cli 為資料庫或是資料表建立不同的權限。"
tags = [
    "CDH","Cloudera","MIT Kerberos","Hadoop",
    "authentication", "security", "Hive", "Sentry"
]
+++

# 如何在 Hive 上面整合 Sentry 與 Kerberos

* 上次文章介紹完 Kerberos 後，此章節為如何在 Hive 上面針對不同的 Table 根據不同的 Principle 建立權限 

* 為什麼需要這個？

若是沒有做好權限的管理，那麼任何一個 User 便可以去對任意的 Table 進行修改，會造成許多問題

* OS : CentOS 7
* 結點數：3
* CM 版本：5.16.2
* CDH 版本：5.16.2
* 叢集資訊：

| Hostname                                                       | IP         |
| -------------------------------------------------------------- | ---------- |
|       all.asia-east1-a.c.fair-terminus-264703.internal all     | 10.140.0.2 |


## 大致流程

1. 進入 Cloudera Manager 更改設定檔
2. 進入 beeline 進行 Table 權限的控管

### 進入 Cloudera Manager ，建立 Hive 臨時用的特權帳號

#### 1. 更改 Hive Warehouse Directory 的權限


```
sudo -u hdfs hdfs dfs -chmod -R 771 /user/hive/warehouse
sudo -u hdfs hdfs dfs -chown -R hive:hive /user/hive/warehouse
```

若是有上 Kerberos 需要使用 kinit hdfs 相關的 Principle

查看權限是否有設定完成：

```
[root@all ~]# hdfs dfs -ls /user/hive/
Found 1 items
drwxrwx--t   - hive hive          0 2020-01-14 04:01 /user/hive/warehouse
```

#### 2. 進入 Cloudera Manager 更改設定檔的設定


如下圖，進入 Hive Service 的 Configuration 

![1](/img/sentry_hive/1.png)

進入設定檔後，搜尋此參數  **HiveServer2 Enable Impersonation**  ，如下圖，更改此設定檔

![2](/img/sentry_hive/2.png)

如下圖，進入 Yarn Service 的 Configuration

![3](/img/sentry_hive/3.png)

進入設定檔後，搜尋此參數 **HiveServer2 Enable Impersonation** ，如下圖，查看是否有 Hive 的使用者，若沒有則進行新增

![4](/img/sentry_hive/4.png)

如下圖，進入 Hive Service 的 Configuration 

![1](/img/sentry_hive/1.png)

進入設定檔後，搜尋此參數  **hadoop.proxyuser.hive.groups**  ，如下圖，更改此設定檔，新增以下三個使用者 **hive hue sentry**

![5](/img/sentry_hive/5.png)

搜尋此參數  **Sentry Service**  ，並且勾選

![6](/img/sentry_hive/6.png)

搜尋此參數  **Enable Stored Notifications in Database**  ，並且勾選

![7](/img/sentry_hive/7.png)

搜尋此參數  **sentry-site.xml**  ，新增以下設定

這個設定蠻重要的，因為當 Hive 上面的 Sentry 設定 enable 後，如果沒有開啟 testing mode 則會沒有帳號可以進去 beeline 進行權限的控管，當使用這權限建好後，即可把此捨定檔案移除掉。

```
<property>
<name>sentry.hive.testing.mode</name>
<value>true</value>
</property>
```

如下圖所示：

![8](/img/sentry_hive/8.png)

如下圖，進入 Sentry Service 的 Configuration

![9](/img/sentry_hive/9.png)

進入設定檔後，搜尋此參數  **Admin Group**  ，如下圖，更改此設定檔，查看是否有一下三個使用者， **hive, spark,  hue**  ，若沒有則新增。

設定完成後，將上述有改動過的地方進行 deploy configuration 和 重啟服務的動作

![10](/img/sentry_hive/10.png)

![11](/img/sentry_hive/11.png)

重啟服務，更新設定檔

![12](/img/sentry_hive/12.png)

![13](/img/sentry_hive/13.png)

![14](/img/sentry_hive/14.png)

![15](/img/sentry_hive/15.png)

![16](/img/sentry_hive/16.png)


### 2. 進入 beeline 進行 Table 權限的控管

上述服務重新啟動完畢後，進入 beeline 進行權限的建立


```
[root@all ~]# beeline -u "jdbc:hive2://all:10000/;principal=hive/all@ETHON.COM"
OpenJDK 64-Bit Server VM warning: ignoring option MaxPermSize=512M; support was removed in 8.0
OpenJDK 64-Bit Server VM warning: Using incremental CMS is deprecated and will likely be removed in a future release
OpenJDK 64-Bit Server VM warning: ignoring option MaxPermSize=512M; support was removed in 8.0
scan complete in 3ms
Connecting to jdbc:hive2://all:10000/;principal=hive/all@ETHON.COM
Connected to: Apache Hive (version 1.1.0-cdh5.16.2)
Driver: Hive JDBC (version 1.1.0-cdh5.16.2)
Transaction isolation: TRANSACTION_REPEATABLE_READ
Beeline version 1.1.0-cdh5.16.2 by Apache Hive
0: jdbc:hive2://all:10000/>
```

執行命令要注意的地方： 

* 指令中的 **all:10000** 為 Hive Server2 的 hostname 和 port 號

* principal= **hive/all@ETHON.COM** 為 hive 的 Principle

#### 建立一個 role 

使用 create role 建立一個最大權限 ( admin ) 的 role

```
0: jdbc:hive2://all:10000/> CREATE ROLE hive_admin;
INFO  : OK
No rows affected (0.2 seconds)
```

查看 role 是否有建立成功

```
0: jdbc:hive2://all:10000/> show roles;
INFO  : OK
+-------------+--+
|    role     |
+-------------+--+
| hive_admin  |
+-------------+--+
1 row selected (0.125 seconds)
```

將最大權限 grant 給這個 role

```
0: jdbc:hive2://all:10000/> GRANT ALL ON SERVER server1 TO ROLE hive_admin WITH GRANT OPTION;
INFO  : OK
No rows affected (0.348 seconds)
```

有了最大權限的 role 後，我們將 **hive** 這個 Group 加入到 hive_admin 這個 role 裡面，代表，凡是在 hive 這個群組裡面的 user ，都擁有 hive_admin 這個 role 的權限

hive 這個群組裡面的 user 是根據 OS 的群組而訂的，如何查詢這個 user 有在哪些 group 裡面 :

```
[root@all ~]# groups hive
hive : hive
```

將 **hive** 這個 Group 加入到 hive_admin 這個 role 裡面：

```
0: jdbc:hive2://all:10000/> GRANT ROLE hive_admin TO GROUP hive;
INFO  : OK
No rows affected (0.206 seconds)
```

#### 將 DB 的權限建立給 使用者 ( User )

最大權限建立完成後，必須要針對不同的 table 去限制什麼人可以查什麼人可以看，或是進行修改

##### 1. 將該 Database 的所有權限，授予給 ethonwu 這個 user 

建立一個 role 

```
0: jdbc:hive2://all:10000/> CREATE ROLE ethon_role;
INFO  : OK
No rows affected (0.179 seconds)
```

將某個 Database 的所有權限給這個 role

```
0: jdbc:hive2://all:10000/> GRANT ALL ON DATABASE sample TO ROLE ethon_role;
INFO  : OK
No rows affected (0.159 seconds)
```

將上面的 Database 的 hdfs 路徑 ( uri ) 的所有權限給這個 role

```
0: jdbc:hive2://all:10000/> GRANT ALL ON URI 'hdfs://user/hive/warehouse/sample.db/' TO ROLE ethon_role;
INFO  : OK
No rows affected (0.096 seconds)
```

最後將這些有規則的 **role** 賦予給 想要的群組，在這邊想要將此權限給 user ethonwu ，user ethonwu 的群組名稱是 ethonwu ，因此群組名稱選用 ethonwu 

```
0: jdbc:hive2://all:10000/> GRANT ROLE ethon_role TO GROUP ethonwu;
INFO  : OK
No rows affected (0.101 seconds)
```

查看是否有建立成功

```
0: jdbc:hive2://all:10000/> show grant role ethon_role;
INFO  : OK
+----------------------------------------+--------+------------+---------+-----------------+-----------------+------------+---------------+-------------------+----------+--+
|                database                | table  | partition  | column  | principal_name  | principal_type  | privilege  | grant_option  |    grant_time     | grantor  |
+----------------------------------------+--------+------------+---------+-----------------+-----------------+------------+---------------+-------------------+----------+--+
| hdfs://user/hive/warehouse/sample.db/  |        |            |         | ethon_role      | ROLE            | *          | false         | 1581490684596000  | --       |
| sample                                 |        |            |         | ethon_role      | ROLE            | *          | false         | 1581490563559000  | --       |
+----------------------------------------+--------+------------+---------+-----------------+-----------------+------------+---------------+-------------------+----------+--+
2 rows selected (0.216 seconds)
```


##### 2. 讓 john 這個 user 在 only_view.test 這張表中只有 SELECT 的權限

建立一個 role

```
0: jdbc:hive2://all:10000/> create role john_role;
INFO  : OK
No rows affected (0.251 seconds)
```

想要讓 john 在 only_view.test 這張 table 上面只有 select 的權限

```
0: jdbc:hive2://all:10000/> GRANT SELECT ON table only_view.test TO ROLE john_role;
INFO  : OK
No rows affected (0.154 seconds)
```

role 建立好後，將此 role 給 john 的 group

```
0: jdbc:hive2://all:10000/> GRANT ROLE john_role TO GROUP john;
INFO  : OK
No rows affected (0.254 seconds)
```

查看 role 

```
0: jdbc:hive2://all:10000/> show grant role john_role;
INFO  : OK
+------------+--------+------------+---------+-----------------+-----------------+------------+---------------+-------------------+----------+--+
|  database  | table  | partition  | column  | principal_name  | principal_type  | privilege  | grant_option  |    grant_time     | grantor  |
+------------+--------+------------+---------+-----------------+-----------------+------------+---------------+-------------------+----------+--+
| only_view  | test   |            |         | john_role       | ROLE            | select     | false         | 1581496676370000  | --       |
+------------+--------+------------+---------+-----------------+-----------------+------------+---------------+-------------------+----------+--+
1 row selected (0.171 seconds)
```

從 privilege 的地方可以看到只有 select ，與上面的 "*" 不一樣

登入 john 帳號，並且取得 principle 進行測試

登入帳號後，取得 principle 

```
[john@all ~]$ kinit john
Password for john@ETHON.COM:
[john@all ~]$ klist
Ticket cache: FILE:/tmp/krb5cc_1002
Default principal: john@ETHON.COM

Valid starting       Expires              Service principal
02/12/2020 08:48:24  02/13/2020 08:48:24  krbtgt/ETHON.COM@ETHON.COM
	renew until 02/19/2020 08:48:24
```

使用 impala-shell 查詢此 table

```
[john@all ~]$ impala-shell
Starting Impala Shell without Kerberos authentication
Opened TCP connection to all.asia-east1-a.c.fair-terminus-264703.internal:21000
Error connecting: TTransportException, TSocket read 0 bytes
Kerberos ticket found in the credentials cache, retrying the connection with a secure transport.
Opened TCP connection to all.asia-east1-a.c.fair-terminus-264703.internal:21000
Connected to all.asia-east1-a.c.fair-terminus-264703.internal:21000
Server version: impalad version 2.12.0-cdh5.16.2 RELEASE (build e73cce22064ef4972312d895bed2cdb8787a4215)
***********************************************************************************
Welcome to the Impala shell.
(Impala Shell v2.12.0-cdh5.16.2 (e73cce2) built on Mon Jun  3 03:32:01 PDT 2019)

The SET command shows the current value of all shell and query options.
***********************************************************************************
[all.asia-east1-a.c.fair-terminus-264703.internal:21000] > select * from only_view.test;
Query: select * from only_view.test
Query submitted at: 2020-02-12 08:49:44 (Coordinator: http://all:25000)
Query progress can be monitored at: http://all:25000/query_plan?query_id=2d4eeb452cfa5f05:178cd73d00000000
+-----+
| num |
+-----+
| 1   |
| 2   |
+-----+
Fetched 2 row(s) in 0.13s
```

因為在 grant 權限的時候，是有 SELECT 的權限的

但是沒有 INSERT 的權限，在這邊測試看看

```
[all.asia-east1-a.c.fair-terminus-264703.internal:21000] > insert into table only_view.test values(999);
Query: insert into table only_view.test values(999)
Query submitted at: 2020-02-12 08:51:26 (Coordinator: http://all:25000)
ERROR: AuthorizationException: User 'john@ETHON.COM' does not have privileges to execute 'INSERT' on: only_view.test
```

這邊就會出現無法 INSERT 的問題，因為沒有權限

其餘的設定方法可以參考這邊：https://docs.cloudera.com/documentation/enterprise/5-14-x/topics/cm_sg_sentry_service.html


