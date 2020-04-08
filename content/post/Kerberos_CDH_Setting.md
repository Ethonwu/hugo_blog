+++
author = "Ethon Wu"
title = "Kerberos CDH Setting"
date = "2020-01-21T18:04:11+08:00"
description = "Cloudera Manager + MIT Kerberos Enable Setting"
tags = [
    "CDH","Cloudera","MIT Kerberos","Hadoop",
    "authentication", "security"
]
+++


* 當 CDH 的叢集服務架設好後，即便有了基本的權限設定，但是在 **安全性** 上面仍然有很多問題，因此 Cloudera Manager 將各個有 Kerberos 功能的 Component 整合在一起，設定方式只要透過 Cloudera Manager 設定即可，並不需要自己生成某個 Component 的 Principal，Cloudera Manager 會自動去處理這些事情，因此方便了許多，不用一個一個 Component 去進行設定。

* 這篇文章為 **如何安裝 Kerberos 並且透過 Cloudera Manager 去整合 CDH 裡面的 Component**，前期的安裝建置過程直接跳過，如果需要安裝建置的文章可以參考這篇（還沒寫）

## OS 架構

- OS : CentOS 7
- 結點數：3
- CM 版本：5.16.2
- CDH 版本：5.16.2
- 叢集資訊：

| Hostname                                                       | IP         |
| -------------------------------------------------------------- | ---------- |
| edgenode.us-central1-a.c.still-toolbox-265706.internal         | 10.128.0.2 |
| all-in-one-node1.us-central1-c.c.still-toolbox-265706.internal | 10.128.0.3 |
| all-in-one-node2.us-central1-b.c.still-toolbox-265706.internal | 10.128.0.4 |

## Kerberos 架構與認證介紹

參考此文章 ( 之後會寫 )

## 架設步驟：
### 1. 安裝 MIT Kerberos
### 2. 設定 MIT Kerberos 的設定檔
### 3. 進入 Cloudera Manager 啟動 MIT Kerberos 認證功能

---

#### 1.安裝 MIT Kerberos ( 在 10.128.0.2 機器上進行安裝 )

1.安裝 MIT Kerberos 所需要套件

```sh
yum install -y krb5-server openldap-clients krb5-workstation krb5-libs
```

#### 2.設定 MIT Kerberos 的設定檔

1.設定設定檔 （若想要更深入了解設定檔的設定方式，參考此文章 ( 之後會寫 ) ）

安裝完套件後，分別有 **3** 個設定檔需要進行設定，分別為：

* krb5.conf : 主要為 MIT Kerberos Server 的基本設定例如網域名稱，realm 名稱... 等

```sh
vim /etc/krb5.conf

Configuration snippets may be placed in this directory as well
includedir /etc/krb5.conf.d/

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 pkinit_anchors = /etc/pki/tls/certs/ca-bundle.crt
 default_realm = ETHON.COM
 #default_ccache_name = KEYRING:persistent:%{uid}

[realms]
ETHON.COM = {
 kdc = edgenode.us-central1-a.c.still-toolbox-265706.internal
 admin_server = edgenode.us-central1-a.c.still-toolbox-265706.internal
}

[domain_realm]
.edgenode.us-central1-a.c.still-toolbox-265706.internal = ETHON.COM
edgenode.us-central1-a.c.still-toolbox-265706.internal = ETHON.COM
```


> a. default_realm 原本是：EXAMLPLE.COM ，將此名稱改成對應的名稱，這邊的範例是使用 ETHON.COM 當作我 Kerberos 的 realm

---

> b. default_ccache_name 此行需要註解，根據 Cloudera 的[文件](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cm_sg_s4_kerb_wizard.html#concept_irl_x5y_l4)

---

> c. realms 和 domain_realm 部分，將原本預設的 EXAMLPLE.COM 改成剛才命名的 ETHON.COM ，定義好 ETHON.COM 後，需要讓機器知道說  ETHON.COM 是誰，因此domain_realm 和 kdc 、admin_server 部分將原本預設的網域名稱更改成機器的 Hostname ，在我的環境 Hostname 為 edgenode.us-central1-a.c.still-toolbox-265706.internal ，因此改成 edgenode.us-central1-a.c.still-toolbox-265706.internal

---


* kadm5.acl : 主要為定義 Priciple 的 **命名規則** ，可以根據命名規則去取決於這個 Priciple 有哪些權限，例如 允許被更改密碼 、允許列出或修改 Priciple 的 Policies

```sh
vim /var/kerberos/krb5kdc/kadm5.acl

*/admin@ETHON.COM       *
```

> 將後面原本的 EXAMLPLE.COM 改成剛才命名的 ETHON.COM ，後面的 * 字號為，只要所建立的 Principle 名稱為 xxxx/admin@ETHON.COM 的話，則這個 Priciple 在 Kerberos 擁有最大的權限，意指可以新增修改刪除其他的 Priciple 帳號，和 root 是同等意思，詳細的權限控管方式可以參考此文章（ 之後會寫 ）


* kdc.conf  : 主要為 MIT Kerberos Server 的第二個設定檔，主要是設定 realm 部分的規則（加密、票據時間...等），以及 Port 號，預設為 88 Port

```sh
vim /var/kerberos/krb5kdc/kdc.conf

[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 ETHON.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab

  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```
---

> realms 部分，將原本預設的 EXAMLPLE.COM 改成剛才命名的 ETHON.COM ，其餘的設定保留不動，如果想要了解更多可以參考此文章（ 之後會寫 ）

---


2.初始化 Kerberos 資料庫

   根據剛剛所設定的 ETHON.COM 名稱，建立 Kerberos 的資料庫

```sh
[root@edgenode ~]# kdb5_util create –r ETHON.COM -s
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'ETHON.COM',
master key name 'K/M@ETHON.COM'
You will be prompted for the database Master Password.
It is important that you NOT FORGET this password.
Enter KDC database master key:
Re-enter KDC database master key to verify:
```
   執行這個命令後，需要設定密碼

3.建立 Kerberos 得特權帳號，以及 Cloudera Manager 的 Kerberos 特權帳號

   因為 Cloudera Manager 需要將為每一個 Component 建立 Kerberos 的 Priciple ，所以需要特權帳號
   Cloudera Manager 的特權帳號

```sh
[root@edgenode ~]# kadmin.local
Authenticating as principal root/admin@ETHON.COM with password.
kadmin.local:  addprinc cloudera-scm/admin
WARNING: no policy specified for cloudera-scm/admin@ETHON.COM; defaulting to no policy
Enter password for principal "cloudera-scm/admin@ETHON.COM":
Re-enter password for principal "cloudera-scm/admin@ETHON.COM":
Principal "cloudera-scm/admin@ETHON.COM" created.
```

   此處需要輸入密碼

   Kerberos 的特權帳號

```sh
kadmin.local:  addprinc ethon/admin
WARNING: no policy specified for ethon/admin@ETHON.COM; defaulting to no policy
Enter password for principal "ethon/admin@ETHON.COM":
Re-enter password for principal "ethon/admin@ETHON.COM":
Principal "ethon/admin@ETHON.COM" created.
```

  此處同上也需要輸入密碼

4.啟動 Kerberos 相關服務，並且設定開機啟動

```sh
[root@edgenode ~]# systemctl start krb5kdc
[root@edgenode ~]# systemctl start kadmin
[root@edgenode ~]# systemctl enable krb5kdc
Created symlink from /etc/systemd/system/multi-user.target.wants/krb5kdc.service to /usr/lib/systemd/system/krb5kdc.service.
[root@edgenode ~]# systemctl enable kadmin
Created symlink from /etc/systemd/system/multi-user.target.wants/kadmin.service to /usr/lib/systemd/system/kadmin.service.
```

5.在另外兩台機器上安裝 Kerberos client 服務與編輯設定檔 ( 另外兩台機器 ，10.128.0.3 , 10.128.0.4)

```sh
yum install -y krb5-libs krb5-workstation
```

   將 10.128.0.2 上面的 /etc/krb5.conf 設定檔傳至另外兩台機器

```sh
ssh root@10.128.0.2
scp /etc/krb5.conf root@10.128.0.3
scp /etc/krb5.conf root@10.128.0.4
```

---

### 3.進入 Cloudera Manager 啟動 MIT Kerberos 認證功能

1.進入 Cloudera Manager -> Administration -> Security 點選 Enable Kerberos

![1](/img/1.png)

![2](/img/2.png)

2.點選 Enable Kerberos 後，全部進行勾選，並且點選 Continue

![3](/img/3.png)

3.編輯 Setup KDC 頁面

![4](/img/4.png)

這邊一個一個欄位進行介紹

KDC Type : 在這邊我們是用 yum 安裝 Kerbreros ，因此是使用 MIT KDC

Kerberos Encryption Types : 這個是 Kerberos 的加密模式，這個設定是根據前面的設定檔而定的，若需要查找可以使用以下命令去找:

```sh
[root@edgenode ~]# cat /var/kerberos/krb5kdc/kdc.conf | grep 'supported_enctypes'
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal
  arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal
  des-cbc-md5:normal des-cbc-crc:normal
```

Kerberos Security Realm : 就是一開始設定 krb5.conf 的 realm ，我的名稱是 ETHON.COM

KDC Server Host & KDC Admin Server Host : Kerberos 的服務架設的 Hostname ，在這邊我是將兩個服務架設在同一台機器上因此是 edgenode.us-central1-a.c.still-toolbox-265706.internal


4.Manage krb5.conf 部分

此部分不用勾選 ，直接下一步，若是勾選，Cloudera Manager 會強制更改 Kerberos 中的內容，導致出錯

![5](/img/5.png)

5.Setup KDC Account 頁面

這邊輸入在前面建立的 cloudera-scm/admin 的特權帳號

![6](/img/6.png)

6.設定成功，Continue 後，進入 Configure Principals 頁面設定每一個 Component 的 Principals ，這邊我是維持預設進行設定
，若是有個人喜好可以自行設定


![7](/img/7.png)

![8](/img/8.png)

7.Configure Ports 頁面 ，這邊 Ports 號就按照預設值，並且打勾 重啟叢集

![9](/img/9.png)

![10](/img/10.png)

![11](/img/11.png)

## 功能測試
1.算 Pi 測試

在我的 OS 裡面有一個帳號為 ethonwu ，在 Kerberos 已經建立好 Principle ，建立 Principle 方式可以參考前面的教學

a. 取得 principle

```sh
[ethonwu@edgenode ~]$ kinit ethonwu
Password for ethonwu@ETHON.COM:
[ethonwu@edgenode ~]$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: ethonwu@ETHON.COM

Valid starting       Expires              Service principal
2020-01-21T08:08:33  2020-01-22T08:08:33  krbtgt/ETHON.COM@ETHON.COM
	renew until 2020-01-28T08:08:33
```

b. 執行 Hadoop 的算 Pi example

```sh
[ethonwu@edgenode ~]$ hadoop jar /opt/cloudera/parcels/CDH/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar pi 10 1
...
Starting Job
20/01/21 08:10:11 INFO client.RMProxy: Connecting to ResourceManager at all-in-one-node1.us-central1-c.c.still-toolbox-265706.internal/10.128.0.3:8032
20/01/21 08:10:11 INFO hdfs.DFSClient: Created token for ethonwu: HDFS_DELEGATION_TOKEN owner=ethonwu@ETHON.COM, renewer=yarn, realUser=, issueDate=1579594211903, maxDate=1580199011903, sequenceNumber=12, masterKeyId=2 on 10.128.0.2:8020
20/01/21 08:10:11 INFO security.TokenCache: Got dt for hdfs://edgenode.us-central1-a.c.still-toolbox-265706.internal:8020; Kind: HDFS_DELEGATION_TOKEN, Service: 10.128.0.2:8020, Ident: (token for ethonwu: HDFS_DELEGATION_TOKEN owner=ethonwu@ETHON.COM, renewer=yarn, realUser=, issueDate=1579594211903, maxDate=1580199011903, sequenceNumber=12, masterKeyId=2)
20/01/21 08:10:12 INFO input.FileInputFormat: Total input paths to process : 10
20/01/21 08:10:12 INFO mapreduce.JobSubmitter: number of splits:10
20/01/21 08:10:12 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1579593927711_0002
20/01/21 08:10:12 INFO mapreduce.JobSubmitter: Kind: HDFS_DELEGATION_TOKEN, Service: 10.128.0.2:8020, Ident: (token for ethonwu: HDFS_DELEGATION_TOKEN owner=ethonwu@ETHON.COM, renewer=yarn, realUser=, issueDate=1579594211903, maxDate=1580199011903, sequenceNumber=12, masterKeyId=2)
20/01/21 08:10:13 INFO impl.YarnClientImpl: Submitted application application_1579593927711_0002
20/01/21 08:10:13 INFO mapreduce.Job: The url to track the job: http://all-in-one-node1.us-central1-c.c.still-toolbox-265706.internal:8088/proxy/application_1579593927711_0002/
20/01/21 08:10:13 INFO mapreduce.Job: Running job: job_1579593927711_0002
...
20/01/21 08:10:24 INFO mapreduce.Job:  map 0% reduce 0%
20/01/21 08:10:32 INFO mapreduce.Job:  map 10% reduce 0%
20/01/21 08:10:37 INFO mapreduce.Job:  map 20% reduce 0%
20/01/21 08:10:41 INFO mapreduce.Job:  map 30% reduce 0%
20/01/21 08:10:46 INFO mapreduce.Job:  map 40% reduce 0%
20/01/21 08:10:50 INFO mapreduce.Job:  map 50% reduce 0%
20/01/21 08:10:54 INFO mapreduce.Job:  map 60% reduce 0%
20/01/21 08:10:58 INFO mapreduce.Job:  map 70% reduce 0%
20/01/21 08:11:02 INFO mapreduce.Job:  map 80% reduce 0%
20/01/21 08:11:07 INFO mapreduce.Job:  map 90% reduce 0%
20/01/21 08:11:11 INFO mapreduce.Job:  map 100% reduce 0%
20/01/21 08:11:16 INFO mapreduce.Job:  map 100% reduce 100%
20/01/21 08:11:16 INFO mapreduce.Job: Job job_1579593927711_0002 completed successfully
...
```

2.進入 Hive 查詢 Table

Kerberos 啟動後，就必須要使用 beeline 進行連線了

a. 首先如同上面一樣，取得 ethonwu 的 Principle

b. 使用 beeline 登入

```sh
beeline -u "jdbc:hive2://all-in-one-node1.us-central1-c.c.still-toolbox-265706.internal:10000/;principal=hive/all-in-one-node1.us-central1-c.c.still-toolbox-265706.internal@ETHON.COM"
```

這邊分兩段來講解：

* HiveServer port : all-in-one-node1.us-central1-c.c.still-toolbox-265706.internal:10000

* Hive Priciple : principal=hive/all-in-one-node1.us-central1-c.c.still-toolbox-265706.internal@ETHON.COM

取得授權後即可連線至 Hive

c. select & count

select：

```sh
0: jdbc:hive2://all-in-one-node1.us-central1-> select * from test;
INFO  : Compiling command(queryId=hive_20200121082626_25b4b03b-c284-4b75-8bdd-b31b231aacd7): select * from test
INFO  : Semantic Analysis Completed
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:test.name, type:string, comment:null), FieldSchema(name:test.id, type:int, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20200121082626_25b4b03b-c284-4b75-8bdd-b31b231aacd7); Time taken: 0.263 seconds
INFO  : Executing command(queryId=hive_20200121082626_25b4b03b-c284-4b75-8bdd-b31b231aacd7): select * from test
INFO  : Completed executing command(queryId=hive_20200121082626_25b4b03b-c284-4b75-8bdd-b31b231aacd7); Time taken: 0.001 seconds
INFO  : OK
+------------+----------+--+
| test.name  | test.id  |
+------------+----------+--+
| test       | 1        |
| ethon      | 2        |
+------------+----------+--+
2 rows selected (0.425 seconds)
```

count：

```sh
0: jdbc:hive2://all-in-one-node1.us-central1-> select count(*) from test;
INFO  : Compiling command(queryId=hive_20200121082727_cc379d31-22e6-4157-842d-488bd7bac57e): select count(*) from test
INFO  : Semantic Analysis Completed
....
INFO  : OK
+------+--+
| _c0  |
+------+--+
| 2    |
+------+--+
1 row selected (29.714 seconds)
```

## 常見問題

1.Hue Ticket renewer

```sh
Couldn't renew kerberos ticket in order to work around Kerberos 1.8.1 issue. Please check that the ticket for ....
```
解決方法：

編輯  /var/kerberos/krb5kdc/kdc.conf

```sh
vim /var/kerberos/krb5kdc/kdc.conf
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 ETHON.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  max_renewable_life = 7d 0h 0m 0s
  default_principal_flags = +renewable
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

新增

```sh
max_renewable_life = 7d 0h 0m 0s
default_principal_flags = +renewable
```

修改 krbtgt 和 hue ticket renewer 的 maxrenewlife

```sh
[root@edgenode ~]# kadmin.local
Authenticating as principal root/admin@ETHON.COM with password.
kadmin.local:  modprinc -maxrenewlife 90day +allow_renewable hue/all-in-one-node2.us-central1-b.c.still-toolbox-265706.internal@ETHON.COM
Principal "hue/all-in-one-node2.us-central1-b.c.still-toolbox-265706.internal@ETHON.COM" modified.
kadmin.local:  modprinc -maxrenewlife 90day krbtgt/ETHON.COM@ETHON.COM
Principal "krbtgt/ETHON.COM@ETHON.COM" modified.
```



