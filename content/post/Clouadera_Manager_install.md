+++
author = "Ethon Wu"
title = "Clouadera Manager Three Install Path"
date = "2019-04-08T15:55:33+08:00"
description = "Three install Cloudera Manager & CDH ways"
tags = [
    "CDH","Cloudera Manager","Hadoop"
]
+++

架設過程是已離線的方式進行安裝，因為有一些環境的叢集是沒有連網的，所以建置方式是以沒有網路的狀態進行安裝。

以下為架設過程的流程圖：


- [系統前置作業](#1)
	- [最佳化 OS 系統設定方式](#1.1)
	- [yum / apt 部分設定](#1.2)
- [資料庫設定](#2)
- [Parcels V.S Package](#3)
- [安裝Cloudera Manager A, B ,C 方式統整](#4)
	- [A方法](#4.1)
	- [B方法](#4.2)
	- [C方法](#4.3)
- [三種方法的總結](#5)
 
<h1 id="1">系統前置作業</h1>

還沒寫

<h2 id="1.1">最佳化 OS 系統設定方式</h2>

還沒寫

<h2 id="1.2"> yum 或 apt 的設定</h2>

有網路狀態安裝套件的方法

無網路狀態安裝套件的方法：

1. 使用CentOS DVD裡面的rpm檔案安裝所需要的服務：

檢查OS是否有讀取到CD的資訊


    ls -al /dev/cdrom


輸出結果

    lrwxrwxrwx. 1 root root 3 3月 7 17:51 /dev/cdrom -> sr0

代表有讀取到光碟內容

在根目錄下建立掛載資料夾

    mkdir ~/<dir name>
    mount -t iso9660 -o ro /dev/cdrom ~/<dir name>

2. 若沒提供CentOS DVD 只能將CentOS DVD的ISO檔移動到叢集內：

將iso檔下載下來後在將檔案移入叢集主機內

    wget http:// URL
    scp <file location> <account>@<IP>:<location>

在根目錄下建立掛載資料夾

    mkdir ~/<dir name>
    mount -t iso9660 -o loop <location> ~/<dir name>

編輯repo檔案

    vi /etc/yum.repo.d/cd.repo

內容為下

    # baseurl 因為目前還沒安裝web service 所以baseurl路徑為file://
    [cd]
    name=cd
    baseurl=file:///<dir path>
    enable=1
    gpgcheck=0

因為我很想用 **vim** 所以急著裝 **vim**

設定完成後，就可以使用 yum install 安裝基本的套件了

    yum -y install httpd

<h1 id="2">資料庫設定與串接</h1>

Cloudera Manager and Managed Service Datastores
[參考網站](https://www.cloudera.com/documentation/enterprise/5-8-x/topics/cm_ig_installing_configuring_dbs.html#cmig_topic_5)

CDH 在運行各個服務的時候，這些服務需要存取一些所需要的資訊，因此必須使用到Database，首先先介紹各個 Database，在介紹各個服務存取在Database上的資訊

 結構化 v.s 非結構化介紹 v.s 半結構化資料
[參考網站1](https://ithelp.ithome.com.tw/articles/10200157)

[參考網站2](https://kevinwang.gitbooks.io/bigdata/content/general/structured-data.html)

1. 結構化資料：已經定義好的欄位資訊，可以很明顯地了解欄位中的意義。

2. 非結構化資料：一團亂的資料，無法明顯地了解欄位中的意義，例如：影音串流。

3. 半結構化資料：介於結構化資料與非結構化資料之間，它不保證一致性。舉例來說，一個資料表中，可能只有部份資料列含有電話欄位，同時只有部份資料列含有地址訊息。
例子（[參考網站2](https://kevinwang.gitbooks.io/bigdata/content/general/structured-data.html)）：

xml
    <users>
    <user>
    <name>Squall</age>
    <age>18</age>
    <weapon>gunblade</weapon>
    </user>
    <user>
    <name>Mario</school>
    <job>plumber</job>
    <shoe_color>brown</shoe_color>
    </user>
    </users>

age 與 job 的部分，此方式的缺點是需要消耗額外的空間。

各個服務所需要存取在DB上的資訊：

1. Oozie Server : 包含 Oozie workflow, coordinator, 和 bundle data 這些資料，儲存空間large
2. Sqoop Server : 包含 connector, driver, links and jobs 這些資料，儲存空間small
3. Activity Monitor : 包含 past activities 的資料，在大型的叢集中，所需要的儲存空間large
4. Reports Manager: 包含 disk utilization 和 processing activities 的資訊，儲存空間medium
5. Hive Metastore Server : 包含 Hive metadata ，儲存空間small
6. Hue Server : 包含 user資訊, job submissions, Hive queries，儲存空間small
7. Sentry Server : 包含 authorization metadata ， 儲存空間small
8. Cloudera Navigator Audit Server : 包含 auditing information ，在大型的從集中，所需要的儲存空間large
9. Cloudera Navigator Metadata Server : 包含 authorization, policies, 和 audit report metadata ，儲存空間small

Database 參考此篇文章

<h1 id="3"> Parcels V.S Package </h1>

[link title](https://www.cloudera.com/documentation/enterprise/5-8-x/topics/cm_ig_managing_software.html)

可以使用兩種方法進行安裝 Package 和 Parcel

* Package 方式 有多個安裝包可提供安裝

* Parcel 方式 類似一個tarbal的方式，把所有的資料包起來，如果遇到版本要更新，隨即更換一個parcel檔即可比起Package的方式來的方便

在 Advantages of Parcels 部份中有提到Puppet自動化部屬工具，有時間可以研究。

[link title](https://shazi.info/puppet-%E8%87%AA%E5%8B%95%E5%8C%96%E9%83%A8%E7%BD%B2-%E5%AE%89%E8%A3%9D%E5%88%9D%E5%A7%8B%E5%8C%96/)

以下為Parcel 版本下載的網址
在架設過程中要拉資料可以參考此網址下載

[link title](https://archive.cloudera.com/cdh5/parcels/)

<h1 id="4"> Installing Cloudera Manager and CDH with Path A, B, C </h1>


以下步驟為設定完成前置作業後，進行cloudera-manager的安裝方式，分為三種：

Path A, B, C

[Local install guide](https://www.cloudera.com/documentation/enterprise/5-8-x/topics/cm_ig_create_local_package_repo.html#cmig_topic_21_3)

安裝方式有三種

<h2 id="4.1"> Path A </h2>

Installation Path A - Automated Installation by Cloudera Manager (Non-Production Mode)

Path A 的安裝方式是從網路上下載一個binary檔，然後執行自動安裝

此方式在文件中闡述不建議用此方式安裝，建議提供展示用途

> This path is recommended for demonstration and proof-of-concept deployments, but is not recommended for production deployments because its not intended to scale and may require database migration as your cluster grows.

[link](https://www.cloudera.com/documentation/enterprise/5-8-x/topics/cm_ig_install_path_a.html#id_vwc_vym_25)

下載檔案方式


    wget https://archive.cloudera.com/cm5/installer/latest/cloudera-manager-installer.bin


讓該檔案變成可執行檔


    chmod u+x cloudera-manager-installer.bin


執行該檔案


    sudo ./cloudera-manager-installer.bin


若是使用Local的方式則要帶--skip_repo_package=1 參數


    sudo ./cloudera-manager-installer.bin --skip_repo_package=1


由於好奇cloudera-manager-installer.bin裡面的架構，於是解開了這隻檔案，該binary檔是由 lua 所撰寫，我是第一次看到此語言
解開方式：


    binwalk cloudera-manager-installer.bin


發現是zip的結構，於是解開它


    mv cloudera-manager-installer.bin cloudera-manager-installer.zip
    unzip cloudera-manager-installer.zip



該binary檔是由 lua 所撰寫，我是第一次看到此語言，程式碼非常多，也不太清楚語言架構，經過了一段時間研究後，程式運行會先偵測OS系統、版本、語系，然後再執行一隻會自動寫入yum、apt設定檔的檔案。

設定檔設定完成後，主程式在根據設定檔給的網址（http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5/），進行安裝套件的動作; 若是安裝在Local端（無對外連線），則在運行binary檔案時需要加入 --skip_repo_package=1 參數，就不會建立repo檔案，直接安裝。

安裝好cloud-manager後，進入網頁進行其他服務的安裝

<h2 id="4.2"> Path B </h2>

Installation Path B - Installation Using Cloudera Manager Parcels or Packages

Path B 的方式是自己手動設定yum、apt的檔案，並且安裝。

設定好yum、apt的路徑後，以下使用yum為例子：

安裝java sdk


    sudo yum install oracle-j2sdk1.7


在Server端安裝Cloudera Manager Server


    sudo yum install cloudera-manager-daemons cloudera-manager-server


注意：如果內部資料庫是使用Oracle database，要使用cloudera-manager必須把  CM_JAVA_OPTS  參數調整成  -Xmx4G 

在其他worker端安裝Cloudera Manager Agent


    sudo yum install cloudera-manager-agent cloudera-manager-daemons


全部安裝完成後，將兩個服務運作起來


    sudo service cloudera-scm-agent start

    sudo service cloudera-scm-server start


<h2 id="4.3"> Path A </h2>

Installation Path C - Manual Installation Using Cloudera Manager Tarballs

Path C 的方式比Path B 、Path C 更加複雜，前兩個的方式基本設定完成後隨即使用yum、apt來進行安裝，安裝過程會把cloudera-manager需要的設定設定完成，但是Path C 方法則是把已經可以執行的cloudera-manager打包成一個tarball，通過下載與基本設定，使其可以運作。

首先先下載tarball檔案

tarball資訊[網址](https://www.cloudera.com/documentation/enterprise/release-notes/topics/cm_vd.html#CM5.14.0)

下載tarball[網址](https://archive.cloudera.com/cm5/cm/5/)

下載tarball


    wget https://archive.cloudera.com/cm5/cm/5/cloudera-manager-<OS and OS version>-cm<version>_x86_64.tar.gz


* 創建clouder-manager目錄，並且解壓縮至該目錄下。


    sudo mkdir /opt/cloudera-manager
    
    sudo tar xzf cloudera-manager*.tar.gz -C /opt/cloudera-manager


新增cloudera-scm使用者帳號


    sudo useradd --system --home=/opt/cloudera-manager/cm-5.6.0/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm


* 建立cloudera-manager-server 儲存資料目錄


    sudo mkdir /var/lib/cloudera-scm-server


給予權限

    sudo chown cloudera-scm:cloudera-scm /var/lib/cloudera-scm-server


設定  server_host  與  server_port  更改  tarball_root/etc/cloudera-scm-agent/config.ini  文件

* 如果要更改log檔案存放路徑 [網址](https://www.cloudera.com/documentation/enterprise/5-8-x/topics/cm_ig_install_path_c.html#cmig_topic_6_7_2__section_yx3_h2w_wm)

* 創建Parcel目錄

Server 端設定路徑

    sudo mkdir -p /opt/cloudera/parcel-repo
    
    sudo chown cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo


Worker 端設定路徑


    sudo mkdir -p /opt/cloudera/parcels
    
    sudo chown cloudera-scm:cloudera-scm /opt/cloudera/parcels


* cloudera-manager-server 服務啟動

因為 Path C 是並不是採用yum、apt的方式安裝，因此路徑是在安裝目錄底下


    sudo <安裝目錄路徑>/etc/init.d/cloudera-scm-server start


可以把該目錄複製到系統的  /etc/init.d/  底下，就可以使用  chkconfig cloudera-scm-agent on  設定開機啟動

* 相依套件的安裝 [參考](https://www.cloudera.com/documentation/enterprise/5-8-x/topics/cm_ig_install_path_c.html#concept_ycd_vtk_rx)

<h1 id="5"> Path A,B,C 差異性 </h1>

依安裝某個套件A為例子

Path A：採自動化安裝，舉例：執行一個腳本，迅速將A套件安裝完畢。

優點：速度fast，適合展示或迅速架設環境。

缺點：無法根據使用者的需求去進行安裝。

Path B：使用 yum 或是 apt 方式進行安裝，舉例：使用套件管理工具(yum, apt)來安裝A套件。

優點：雖然要設定yum、apt檔案，但是比起Path A 可以根據使用者需求去進行架設。

缺點：速度medium，過程中需要仰賴套件管理工具(yum, apt,...)來安裝，如果遇到比較特殊的系統就無法使用此方法。

Path C：下載tarball方式，再將cloudera-manager-server所需要的環境配置手動配置好，舉例：去官方網站下載A套件的Source code並且進行編譯。

優點：適合沒有套件管理工具的OS，一步一步完成安裝部署。

缺點：速度slow，設定步驟繁雜，除非該OS沒有套件管理工具，則才使用此方案。
