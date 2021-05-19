+++
author = "Ethon Wu"
title = "透過 Packetbeat 來監控 Elasticsearch 查詢記錄"
date = "2020-08-5T18:04:11+08:00"
description = "透過 Packetbeat 來監控 Elasticsearch 查詢記錄"
tags = [
    "audit","Elasticsearch","Packetbeat","Query",
    "Monitoring"
]
+++


# 透過 Packetbeat 來監控 Elasticsearch 查詢記錄

* 最近在摸索 Elastic 的一些工具，因緣際會看到了有一個 beat 叫做 **Packetbeat** ，他可以去監控網路有關的資料，並且將其送到 logstash ，最後轉發到 Elasticsearch 上，透過 Kibana 做一些可視化的應用，於是找到了官方有寫得一篇文章： [Monitoring the Search Queries](https://www.elastic.co/blog/monitoring-the-search-queries) ，雖然是 2016 的文章，但是看完文章後，覺得很有興趣，因此寫了這篇文章來實作他裡面的步驟以及細節。

# 架構以及運作過程

* 首先我們會有一個 Elasticsearch 的正式環境，並且會有一些 Application 會去定期的對 Elasticsearch 叢集打 Search 的 API 做查詢，然後我們透過 Packetbeat 去監聽 Application 送過去 正式環境 Elasticsearch 叢集的封包，Packetbeat 再將封包送去 Logstash 進行處理，將有 Search API  相關的封包整理出來，送到 監控專用的 Elasticsearch 上，如下圖所示:

![1](/img/packetbeat_query_dashboard/packetbeat_arch.png)

# 架設監控 Elasticsearch 查詢記錄所需環境

* 這邊的環境是根據已經有 Elasticsearch 和 Kibana 環境，因此不會介紹如何運行 Elasticsearch 和 Kibana 

## 環境介紹 

* OS : CentOS7
* ES Cluster : 2 ( 一個為主要的 ES ，另一個為存放監控資料得 ES )
* ELK : 6.4.3

| Hostname  | IP             | Description                                      |
| --------- | -------------- | ------------------------------------------------ |
| es1       | 192.168.30.171 | 主要的 ES 中的 Ingest node 在這邊安裝 Packetbeat |
| es4       | 192.168.30.174 | Monitoring 用的 Logstash                         |
| localhost | 192.168.30.39  | Monitoring 用的 Elasticsearch & Kibana           |

* 這邊使用 rpm 安裝的原因是比起 tarball 的方式，比較好管理版本，以及 rpm 透過 systemd 去管理，比較不用在寫啟動的腳本
- Packetbeat 6.4.3 [下載頁面](https://www.elastic.co/downloads/past-releases/packetbeat-6-4-3)
- Logstash 6.4.3 [下載頁面](https://www.elastic.co/downloads/past-releases/logstash-6-4-3)
- 根據上面網站內容下載 rpm 套件並且進行安裝


#### 1. 安裝 Packetbeat 所需套件

進入 es1 安裝 Packetbeat 和相依套件


    yum install libpcap -y
    wget https://artifacts.elastic.co/downloads/beats/packetbeat/packetbeat-6.4.3-x86_64.rpm
    rpm -ivh packetbeat-6.4.3-x86_64.rpm


進入 es4 安裝 Logstash


    wget https://artifacts.elastic.co/downloads/logstash/logstash-6.4.3.rpm
    rpm -ivh logstash-6.4.3.rpm


#### 2. 設定 Logstash 的設定檔

* 使用 rpm 安裝的 Logstash 目錄資訊：
 
  設定檔路徑在 /etc/logstash/ 底下

  bin 檔放在   /usr/share/logstash/ 底下

  log 檔放在   /var/log/logstash/ 底下

---

* 這邊主要是接收來自 Packetbeat 傳送過來的資料，並且解析資料取有 search API 的資料

    
    cat /etc/logstash/conf.d/sniff_search.conf
    input {
      beats {
        port => 5044
      }
    }
    filter {
      if "search" in [request]{
        grok {
          match => { "request" => ".*\n\{(?<query_body>.*)"}
        }
        grok {
          match => { "path" => "\/(?<index>.*)\/_search"}
        }
        if [index] {
        }
        else {
          mutate {
            add_field  => { "index" => "All" }
          }
        }
        mutate {
          update  => { "query_body" => "{%{query_body}" }
        }
      }
    }
    output {
      if "search" in [request] and "ignore_unmapped" not in [query_body]{
        elasticsearch {
          hosts => "192.168.30.39:9200"
          index => "es-search-query-%{+YYYY.MM}"
        }
      }
    }



* input 的部分，開啟 listen 的 Port，後面設定 Packetbeat 時，指定到 Logstash 的 Port 要一致
* filter 的部分，主要提取 search 相關的 request 
* output 的部分，需要設定 Monitoring 用的 Elasticsearch 的 Ingest node 的 IP 、Port 和 index 名稱 

--- 

設定完成後，透過 systemd 把服務啟動


    systemctl start logstash
    systemctl enable logstash


查看 log 檔，看服務有沒有順利運作


    root@es4:~# tailf /var/log/logstash/logstash-plain.log
    [2021-05-16T03:43:12,342][WARN ][logstash.outputs.elasticsearch] Detected a 6.x and above cluster: the `type` event field won't be used to determine the document _type {:es_version=>6}
    [2021-05-16T03:43:12,384][INFO ][logstash.outputs.elasticsearch] New Elasticsearch output {:class=>"LogStash::Outputs::ElasticSearch", :hosts=>["//192.168.30.39:9200"]}
    [2021-05-16T03:43:12,418][INFO ][logstash.outputs.elasticsearch] Using mapping template from {:path=>nil}
    [2021-05-16T03:43:12,443][INFO ][logstash.outputs.elasticsearch] Attempting to install template {:manage_template=>{"template"=>"logstash-*", "version"=>60001, "settings"=>{"index.refresh_interval"=>"5s"}, "mappings"=>{"_default_"=>{"dynamic_templates"=>[{"message_field"=>{"path_match"=>"message", "match_mapping_type"=>"string", "mapping"=>{"type"=>"text", "norms"=>false}}}, {"string_fields"=>{"match"=>"*", "match_mapping_type"=>"string", "mapping"=>{"type"=>"text", "norms"=>false, "fields"=>{"keyword"=>{"type"=>"keyword", "ignore_above"=>256}}}}}], "properties"=>{"@timestamp"=>{"type"=>"date"}, "@version"=>{"type"=>"keyword"}, "geoip"=>{"dynamic"=>true, "properties"=>{"ip"=>{"type"=>"ip"}, "location"=>{"type"=>"geo_point"}, "latitude"=>{"type"=>"half_float"}, "longitude"=>{"type"=>"half_float"}}}}}}}}
    [2021-05-16T03:43:12,509][INFO ][logstash.outputs.elasticsearch] Installing elasticsearch template to _template/logstash
    [2021-05-16T03:43:13,254][INFO ][logstash.inputs.beats    ] Beats inputs: Starting input listener {:address=>"0.0.0.0:5044"}
    [2021-05-16T03:43:13,291][INFO ][logstash.pipeline        ] Pipeline started successfully {:pipeline_id=>"main", :thread=>"#<Thread:0xf7d1d38 run>"}
    [2021-05-16T03:43:13,360][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}
    [2021-05-16T03:43:13,472][INFO ][org.logstash.beats.Server] Starting server on port: 5044
    [2021-05-16T03:43:13,791][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}


#### 3. 設定 Packetbeat 的設定檔

* 使用 rpm 安裝的 Packetbeat 目錄資訊：

  設定檔路徑在 /etc/packetbeat/ 底下

  bin 檔放在   /usr/share/packetbeat/ 底下

  log 檔放在   /var/log/packetbeat/ 底下

---

* 編輯 /etc/packetbeat/packetbeat.yml 設定檔，指定監聽的網卡以及 Port 號


    root@es1:~# cat /etc/packetbeat/packetbeat.yml | grep -v "#\|^$"
    packetbeat.interfaces.device: eth0
    packetbeat.flows:
      timeout: 30s
      period: 10s
    packetbeat.protocols:
    - type: http
      ports: [9200]
      send_request: true
      include_body_for: ["application/json", "x-www-form-urlencoded"]
    setup.template.settings:
      index.number_of_shards: 1
    setup.kibana:
      host: "192.168.30.39:5601"
    output.logstash:
      hosts: ["192.168.30.174:5044"]


這邊使用 grep 將不必要的註解與空格過濾掉
* packetbeat.interfaces.device 這邊主要是設定你要 **監聽** 的網卡名稱，可以透過 ip addr 或是 ifconfig 查看
* packetbeat.protocols 在 **原始** 的設定檔中，有很多範例，不過我們只要做監聽 HTTP 的協定，因此可以將其他的協定助解掉，並且額外新增 send_request 和 include_body_for 的參數
* output.elasticsearch 因為是丟到 logstash 做過濾，因此可以將此註解

---

設定完成後，透過 systemd 把服務啟動

    systemctl start packetbeat
    systemctl enable packetbeat


查看 log 檔，看服務有沒有順利運作

    root@es1:~# tailf /var/log/packetbeat/packetbeat
    2021-05-16T04:13:43.050Z	INFO	instance/beat.go:383	packetbeat start running.
    2021-05-16T04:13:43.050Z	INFO	[monitoring]	log/log.go:114	Starting metrics logging every 30s
    2021-05-16T04:13:45.563Z	INFO	pipeline/output.go:95	Connecting to backoff(async(tcp://192.168.30.174:5044))
    2021-05-16T04:13:45.563Z	INFO	pipeline/output.go:105	Connection to backoff(async(tcp://192.168.30.174:5044)) established
    2021-05-16T04:14:13.053Z	INFO	[monitoring]	log/log.go:141	Non-zero metrics in the last 30s	{"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":80,"time":{"ms":88}},"total":{"ticks":370,"time":{"ms":378},"value":370},"user":{"ticks":290,"time":{"ms":290}}},"info":{"ephemeral_id":"27c5becc-8973-4f98-b7af-559227ca1e91","uptime":{"ms":30015}},"memstats":{"gc_next":39275776,"memory_alloc":28508712,"memory_total":50686008,"rss":54685696}},"libbeat":{"config":{"module":{"running":0}},"output":{"events":{"acked":173,"batches":13,"total":173},"read":{"bytes":78},"type":"logstash","write":{"bytes":36877}},"pipeline":{"clients":2,"events":{"active":7,"published":180,"retry":4,"total":180},"queue":{"acked":173}}},"system":{"cpu":{"cores":2},"load":{"1":0.05,"15":0.05,"5":0.03,"norm":{"1":0.025,"15":0.025,"5":0.015}}}}}}
    2021-05-16T04:14:43.052Z	INFO	[monitoring]	log/log.go:141	Non-zero metrics in the last 30s	{"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":130,"time":{"ms":43}},"total":{"ticks":550,"time":{"ms":181},"value":550},"user":{"ticks":420,"time":{"ms":138}}},"info":{"ephemeral_id":"27c5becc-8973-4f98-b7af-559227ca1e91","uptime":{"ms":60015}},"memstats":{"gc_next":39395904,"memory_alloc":26895448,"memory_total":84383056,"rss":7766016}},"libbeat":{"config":{"module":{"running":0}},"output":{"events":{"acked":268,"batches":16,"total":268},"read":{"bytes":96},"write":{"bytes":50423}},"pipeline":{"clients":2,"events":{"active":7,"published":268,"total":268},"queue":{"acked":268}}},"system":{"load":{"1":0.03,"15":0.05,"5":0.03,"norm":{"1":0.015,"15":0.025,"5":0.015}}}}}}
    2021-05-16T04:15:13.052Z	INFO	[monitoring]	log/log.go:141	Non-zero metrics in the last 30s	{"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":160,"time":{"ms":33}},"total":{"ticks":730,"time":{"ms":180},"value":730},"user":{"ticks":570,"time":{"ms":147}}},"info":{"ephemeral_id":"27c5becc-8973-4f98-b7af-559227ca1e91","uptime":{"ms":90015}},"memstats":{"gc_next":37809696,"memory_alloc":21246240,"memory_total":115570856}},"libbeat":{"config":{"module":{"running":0}},"output":{"events":{"acked":277,"batches":14,"total":277},"read":{"bytes":84},"write":{"bytes":48730}},"pipeline":{"clients":2,"events":{"active":7,"published":277,"total":277},"queue":{"acked":277}}},"system":{"load":{"1":0.02,"15":0.05,"5":0.02,"norm":{"1":0.01,"15":0.025,"5":0.01}}}}}}
    2021-05-16T04:15:43.052Z	INFO	[monitoring]	log/log.go:141	Non-zero metrics in the last 30s	{"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":190,"time":{"ms":27}},"total":{"ticks":910,"time":{"ms":180},"value":910},"user":{"ticks":720,"time":{"ms":153}}},"info":{"ephemeral_id":"27c5becc-8973-4f98-b7af-559227ca1e91","uptime":{"ms":120015}},"memstats":{"gc_next":39561632,"memory_alloc":20324400,"memory_total":149361064}},"libbeat":{"config":{"module":{"running":0}},"output":{"events":{"acked":275,"batches":16,"total":275},"read":{"bytes":96},"write":{"bytes":50418}},"pipeline":{"clients":2,"events":{"active":7,"published":275,"total":275},"queue":{"acked":275}}},"system":{"load":{"1":0.01,"15":0.05,"5":0.02,"norm":{"1":0.005,"15":0.025,"5":0.01}}}}}}
    2021-05-16T04:16:13.052Z	INFO	[monitoring]	log/log.go:141	Non-zero metrics in the last 30s	{"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":220,"time":{"ms":37}},"total":{"ticks":1060,"time":{"ms":154},"value":1060},"user":{"ticks":840,"time":{"ms":117}}},"info":{"ephemeral_id":"27c5becc-8973-4f98-b7af-559227ca1e91","uptime":{"ms":150016}},"memstats":{"gc_next":39586048,"memory_alloc":35451432,"memory_total":183189984}},"libbeat":{"config":{"module":{"running":0}},"output":{"events":{"acked":281,"batches":16,"total":281},"read":{"bytes":96},"write":{"bytes":50740}},"pipeline":{"clients":2,"events":{"active":5,"published":279,"total":279},"queue":{"acked":281}}},"system":{"load":{"1":0.01,"15":0.05,"5":0.02,"norm":{"1":0.005,"15":0.025,"5":0.01}}}}}}
    2021-05-16T04:16:43.052Z	INFO	[monitoring]	log/log.go:141	Non-zero metrics in the last 30s	{"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":260,"time":{"ms":32}},"total":{"ticks":1250,"time":{"ms":178},"value":1250},"user":{"ticks":990,"time":{"ms":146}}},"info":{"ephemeral_id":"27c5becc-8973-4f98-b7af-559227ca1e91","uptime":{"ms":180015}},"memstats":{"gc_next":39599648,"memory_alloc":33983304,"memory_total":218398968,"rss":1232896}},"libbeat":{"config":{"module":{"running":0}},"output":{"events":{"acked":275,"batches":17,"total":275},"read":{"bytes":102},"write":{"bytes":50391}},"pipeline":{"clients":2,"events":{"active":5,"published":275,"total":275},"queue":{"acked":275}}},"system":{"load":{"1":0,"15":0.05,"5":0.01,"norm":{"1":0,"15":0.025,"5":0.005}}}}}}


#### 4. 進入 Monitoring 用的 Kibana 查看 或是使用 Rest 對 Elasticsearch 做查詢 有沒有資料進來


* Kibana 查看有沒有資料

![2](/img/packetbeat_query_dashboard/kibana_data.png)

或是

* 對 Elasticsearch 做查詢

    root@es1:~# curl http://192.168.30.39:9200/es-search-query-2021.05/_count | jq .
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100    74  100    74    0     0  18412      0 --:--:-- --:--:-- --:--:-- 24666
    {
      "count": 7839,
      "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
      }
    }


#### 5. 驗證資料是否有被監聽進去 Monitor ES 內

* 簡單的對 es1 上面的 train index 查詢一筆資料


    root@es1:~# curl -XGET "http://192.168.30.171:9200/train/_search" -H 'Content-Type: application/json' -d'
    > {
    >   "query": {
    >     "match_all": {}
    >   },
    >   "size": 1
    > }' | jq .
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100   347  100   293  100    54  53486   9857 --:--:-- --:--:-- --:--:-- 58600
    {
      "took": 3,
      "timed_out": false,
      "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
      },
      "hits": {
        "total": 4,
        "max_score": 1,
        "hits": [
          {
            "_index": "train",
            "_type": "_doc",
            "_id": "2",
            "_score": 1,
            "_source": {
              "id": "data2",
              "post_date": "2019-05-27T17:25:30",
              "context": "love dog"
            }
          }
        ]
      }
    }


* 進入 Monitor Kibana 做查詢

![4](/img/packetbeat_query_dashboard/kibana_query_result.png)


#### 6. 報表以及 Dashboard 的呈現

* 根據之前的參考[文章](https://www.elastic.co/blog/monitoring-the-search-queries)，雖然他有在 [Github Gist](https://gist.github.com/JalehD/d85f85a1aa3bbe534bf0aec0a9c69777) 提供 Visualization 和 Dashboard ，**但是由於 Kibana 的版本不相容，因此無法匯入**， 因此我在這邊從新繪製與參考文章的 Visualization 和 Dashboard。

* 這是舊版在文章上面的舊 Dashboard 根據此圖，畫在 6 版的 Kibana 上

![5](https://api.contentstack.io/v2/assets/57ac7c03bc5ccd727e689321/download?uid=blta05c984567f25af5)

* 有分成四張 Visualization 

##### 1. number-of-searches-per-index

* 使用 Data Table
* Metrics 的部分使用 Count 
* Buckets 的部分使用了兩層
  * 第一層用 Date Histogram 取 @timestamp 欄位
  * 第二層用 Terms 取 query_body.keyword 欄位

![6](/img/packetbeat_query_dashboard/number-of-searches-per-index.png)

##### 2. count-of-query-per-client

* 使用 Pie
* Metrix 的部分使用 Count 
* Buckets 的部分使用了兩層
  * 第一層用 Terms 取 index.keyword 欄位
  * 第二層用 Terms 取 client_ip.keyword 欄位

![7](/img/packetbeat_query_dashboard/count-of-query-per-client.png)

* 可以在 Options 中勾選 Show Labels 顯示 Client IP 

![8](/img/packetbeat_query_dashboard/count-of-query-per-client_lable.png)

##### 3. responsetime-graph

* 使用 Line
* Metrix 的部分使用 Average 取 responsetime 欄位
* Buckets 的部分使用了一層
  * 第一層用 Date Histogram 取 @timestamp 欄位取 2h 

![8](/img/packetbeat_query_dashboard/responsetime-graph.png)

##### 4. Max-Responsetime-per-Query

* 使用 Data Table
* Metrix 的部分使用 Max
* Buckets 的部分使用了一層
  * 第一層用 Trems 取 query_body.keyword 前五筆

![9](/img/packetbeat_query_dashboard/Max-Responsetime-per-Query.png)


* 將四張 Visualization 顯示在 Dashboard 上

![10](/img/packetbeat_query_dashboard/query-analysis.png)

