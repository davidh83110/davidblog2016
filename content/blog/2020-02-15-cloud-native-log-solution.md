---
author: David
categories:
- AWS
- Architecture
comments: true
date: "2020-02-15T00:00:00Z"
modified: "2020-02-15"
published: true
title: Cloud-Native LOG 儲存以及分析方案
url: /aws/系統建置/2020/02/15/cloud-native-log-solution.html
---

隨著 Microservices 的蓬勃發展，從以前的一體式架構慢慢演變成了現今的動輒數十數百個服務同時在線，Log 的量也以很可怕的速度成長中，面對巨量的 Log ，要存在哪裡、要怎麼去利用這些資料反而是現今架構上的難題。 

利用這些 Log 我們可以輕易的去提早發現服務的異常，也可以拿來做分析，產出更珍貴的資源，像是 BI。 
最重要的是可以協助 Debug, 除錯, 還有做 Audit. 

> 我認為 Logging 就是一件很容易被大家認為不重要的事，但他其實很重要，也是很珍貴的資產，絕對值得我們花時間好好處理以及規劃。 

<br />

## **格式問題**
---
一般來說都會是 JSON, 不管你要存存什麼格式 `統一就好` 然後避免直接存 Raw Data。  

Log Level 也是要注意，盡量把 `CRITICAL` / `ERROR` / `WARNING` / `INFO` ...這些分出來。 

<br />

## **要存在哪裡**
---
這是一個很難的問題，市面是常見的方案有以下幾種 
 
A. Elasticsearch + Fluentd(Logstash) + Kibana (ELK) <br />
B. CloudwatchLogs <br />
C. S3 <br />
D. Database <br />
E. Splunk <br />
F. Graylog <br />

<br />

#### A. **ELK**
這套適合成長中的組織，有足夠人力也有足夠彈性可以去維護。  
但是他是 Near-Real-Time 的服務（畢竟中間還要經過 Fluentd），所以會有一定的延遲。 

Pros: <br />
a. 搜尋速度很快 <br />
b. 語法簡單又可以支援比較複雜的查詢語法 <br />
c. UI 平易近人，產出 Charts 也很容易 <br />
d. Logstash / Fluentd 可以預先過濾資訊 <br />
e. 這套太經典，會用的人很多 <br />


Cons: <br />
a. AWS Elasticsearch 太貴，自己架？ okay 那你就會需要去維護它，會有點麻煩。 <br />
b. 權限管理不好做 <br />
c. 關聯分析弱 <br />
d. 你需要同時部署三套服務 <br />
e. Near-Real-Time 

<br />

#### B. **CloudwatchLogs**
這套適合剛起步的團隊，什麼都不用管就丟進去就對了。 

Pros: <br />
a. 簡單暴力 <br />
b. 搭配 Insights 可以有類似 ELK 的效果 <br />


Cons: <br />
a. 語法不太平易近人 <br />
b. 搜尋 raw data 的話，真的很慢 <br />
c. 彈性太低，也不支援 Plugin，只能接 Lambda 做一些事 

<br />

#### **S3**
S3 + Athena 適合做 Access Logs / Networking Traffic Logs 這些比較不常用的 Logs 的查詢以及分析，比較不適合需要一直去看的 Application Logs。 

Pros: <br />
a. 簡單便宜 <br />
b. 可以搭配 Athena 做 Access Logs 的分析 <br />


cons: <br />
a. 基本上沒有查詢的功能 <br />
b. Log 很難找

<br />

#### **Database**
這比較適合已經過濾過的資料，再塞回 Database 會比較好，如果 raw data 直接進 DB <br />
hm..... 可能會很亂也很浪費，如果你又用 NoSQL DB, 那就又更亂了。

<br />

#### **Spplunk**
這是一套稱霸業界的方案，語法多元，圖表清晰，要什麼都做的到。 <br />

Pros: <br />
a. 強大暴力<br />
b. Plugin 也支援<br />
c. 圖表已經太強大到變態的程度了， BI 也可以利用這個來做圖表<br />
d. Real Time<br />
e. 權限控管<br />


Cons: <br />
a. 貴到不行的授權<br />

<br />

#### **Graylog**
其實這是一套需要自己架的方案，但強度堪比 Splunk  
想省錢找CP值高的，就這個了！  

一直很心動，但還沒有時間去試試看 

Pros: <br />
a. 大概跟 Splunk 差不多<br />
b. 免費<br />


Cons: <br />
a. 要自己管理

<br />

## **評比**
---
其實不同工具都有他各自適合的應用場景，`從來都沒有誰最好，只有誰最適合`。

小團隊: CloudwatchLogs

中型團隊: Graylog, ELK

公司很想花錢的： Splunk

Access Logs: S3

想做資料分析 / Big Data: S3 + Athena, Splunk, Graylog

<br />

## **結語**
---

如果要我選，我會想試試看 Graylog 
聽起來很 Fantasy.

詳情怎麼運作的, 怎麼維護就不說了，因為不在本章範圍。

除了工具要好用，要省錢，別忘了導入時的`溝通成本`以及事後的`學習成本`，也會是一個很大的考量。（老闆以及同事接不接受）

<br />
<br />
<div style="text-align: right;">
2020-02-15 12:43 , David in Taipei</div>

<br />
<br />
<br />
