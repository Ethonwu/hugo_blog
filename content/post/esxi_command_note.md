+++
author = "Ethon Wu"
title = "ESXi 指令筆記"
date = "2021-01-08T15:34:33+08:00"
description = "ESXi 指令筆記"
tags = [
    "ESXi","VMWare"
]
+++


公司的 Server 是使用 **免費版** 的 ESXi 系統，無法使用一些工具進行自動化，因此只好默默的打開 ssh port ，好好得研究一下命令
這邊主要做一些在這期間時常使用到的 ESXi 相關的指令，做個筆記


---

### 基本介紹

ESXi 預設不會將 ssh port 打開，必須要透過 WEB 或是 終端畫面進行 Enable ，如下圖啟用方式：

![1](http://0.0.0.0:12500/esxi_note/esxi_enable_ssh.png)

啟用之後，就可以透過 22 port 連線到 ESXi Server 內執行裡面的命令取代 WEB UI

### 免密碼登入

可以自動化就會想要 **免密碼登入** ，讓自動化更齊全嘛！！

將自己的 ssh 公鑰放置在 ESXi 上
這邊假定你已經會產 key 了
登入 ESXi Server 進入目錄編輯 authorized_keys 檔案

```
# 輸入你自己的 ssh 公鑰
vi /etc/ssh/keys-root/authorized_keys
```

### 使用者相關

* 建立一個一般使用者

```
esxcli system account add -i <User Name> -d '<User Description>' -p <Password> -c <Double type password>
```

* -i 的部分後面輸入使用者名稱
* -d 對這個使用這做一些介紹
* -p 輸入使用者登入密碼
* -c 重複輸入設定的密碼







