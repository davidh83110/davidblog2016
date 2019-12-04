---
published: true
layout: post
title: AWS Solutions Architect Associate (SAA) - [PASSED] 心得以及攻略
author: David
modified: 2019-12-05
categories: [AWS]

  
comments: true
---

## AWS Solutions Architect Associate (SAA) - [PASSED]
---

![Alt text](https://live.staticflickr.com/65535/49038426206_83d9c537d2_h.jpg)
<br />


經過了幾週的準備，終於通過了SAA的考試 (82% PASS)。以此紀錄以及分享一下準備的過程。
  


## 事前準備
___

先自我介紹，我從2016年開始接觸AWS，有使用過的服務有：
 - EC2 (EBS, EC2 RI), LB, ASG
 - VPC (subnet, Route Table, Peering connection)
 - ECS / OpsWorks / Beanstalk
 - S3
 - Lambda
 - Cloudwatch / CloudwatchLogs

其餘沒有列出來的服務都只有接觸過並沒有很深入了解。

一開始先閱讀 AWS 官方的考試指南是一定要的。

[考試指南連結](https://d1.awsstatic.com/training-and-certification/docs-sa-assoc/AWS_Certified_Solutions_Architect_Associate-Exam_Guide_EN_1.8.pdf)

裡面明確指出，你會需要閱讀白皮書(WhitePapers)以及了解何謂良好架構，講白話點就是你要知道每個服務的Best Practices以及多個常見的服務組合該怎麼使用，會有哪些限制。

考這些重要嗎？ 坦白說我覺得很重要。 不一定要把所有服務細節都記在腦海裡，但一定要知道哪些服務怎麼組合，使用情境為何，會遇到哪些限制，以及怎麼做規劃。 

有這些知識之後再去做架構規劃會比每次都瞎Google一通好太多了。

`那到底考試都考哪些？`

我個人認為必考的如下
 - EC2 (RI, Placement Group, EBS)
 - ALB / ELB / NLB
 - IAM
 - VPC (NAT / IGW / Subnets / Endpoints / VPN)
 - S3 (大重點！) (Storage Gateway/S3各種種類/Lifecycle)
 - RDS (Aurora / RDS) / RedShift / DynamoDB
 - 如何與 On Premise 的叢集做混合雲架構

最後面會再細講到底考了哪些，以及懶人包。


講了這麼多廢話，下面開始介紹我如何準備這個考試。



## Stage1. 線上課程
---

我第一個使用的是 A Cloud Guru.

[A Cloud Guru - Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

Udemy 上常常會有優惠，或上網可以找到優惠代碼，如果有要購買的人可以稍微找一下，省點錢。畢竟考試很貴。

A Cloud Guru 好用嗎？ 

我覺得如果你從0開始學習AWS，那這將會是一堂不錯的入門課程，還附有隨堂測驗跟Lab。

但如果已經具備一些 AWS 基本知識，那可以直接去做每個項目的隨堂測驗 (e.g. RDS - quiz)。 做完基本就知道自己對這個服務掌握度到哪裡，不足的話，`非常建議把整堂課重看一次。`

最後會有 Mega Test，也就是 120min 模擬真實考試的60題測驗，`A Cloud Guru 的難度比真實測驗要高得太多。` 如果想挫挫自己銳氣或是想完整掌握知識，非常建議去做，但是會有點崩潰。



## Stage2. AWS FAQ
---

之後我發現 A Cloud Guru 還是漏講了很多東西，也或許是 AWS 更新的太快。 我又跑去看所有會考的服務的FAQ。 `獲益良多`

這邊不多說，大家都知道哪些特別需要看。 請見上面前情提要。

[AWS FAQ](https://aws.amazon.com/tw/faqs/)



## Stage3. WhizLabs
---

這是一個線上模擬題庫，我不得不說命中率有75%以上。 非常神準。

裡面包含 8-10 組題庫，還能針對錯誤的題目做客製化題庫，特別拉出來練習，是個考前幾週補強觀念的好地方。 

非常推薦考前兩週開始做，到考試日期剛好差不多作完。

特別要說的是， `Whizlabs 的題目詳解，真的寫得很好`，清楚的告訴你正確觀念，以及你選的為什麼會是錯的。 我覺得大推！

[Whizlabs - AWS SAA C01](https://www.whizlabs.com/aws-solutions-architect-associate/)



## 正式考試
---

正式考試不外乎難度比任何練習題都還要低一點，其實不難答題。

要掌握的觀念就跟題目做久了一樣，就會發現就是那些東西在重複而已。

- EC2
  - Reserved Instances (RI)
  - Placement Group
  - AMI (跨Region如何使用？ 搬過去！)
  - EIP 如何收費 (綁定ENI後不收費)
  - EBS (各種情境Type怎麼選)
  - ASG (Scaling Policies / Terminated Policy / ASG Config)
  - `Use IAM Role instead of insert Access Key`

- VPC
  - Subnets & AZ 的關係 (一個subnet會在一個AZ)
  - Security v.s. Network ACL (`Stateful v.s. Stateless`) (服務之間做Port限制都是SG, ACL不能做Port限制，因為所在的OSI Network Layer不同(Layer6 v.s. Layer4))
  - Routing Table
  - Public Subnet v.s. Private Subnet (`Attched IGW v.s. NAT`)
  - NAT (`要放在Public Subnet, 但是Private Subnet Attach.`，在Private Subnet內的都透過NAT跟外網溝通，NAT內的網路流量`只出不進`)
  - IGW (Public Subent Attach，直接與外網溝通，網路可進可出)
  - Peering Connection (可跨Account)
  - VPN (如何與On Premise溝通)
  - VPC Endpoints

- S3
  - S3 IA / Single IA / Glacier (`Lifecycle`)
  - S3 Policy & IAM 使用情境
  - Cross Region Replication (CORS) & Versioning
  - VPC Endpoints for using internal networking

- RDS 
  - DynamoDB
  - RDS Engines
  - Aurora (MySQL / PSSQL)
  - `Read-Replica & Mutil-AZ` 
  - Cross Region Read-Replica
  - `Mantainence Window & Buckup`
  - Redshift is a Data Warehouse

- IAM
  - Organization
  - IAM User (Access Key & Password) & Group
  - Role & PPolicy
  - Cross Account & sts Roles

- Others
  - 合規類型的 (VPC Flow logs / API Call要去CloudTrails查詢)
  - AWS EMR
  - SSM Parameter Store & Secret Manager
  - Beanstalk & Opsworks
  - ECS
  - Lambda & APIGW
  - Cloudwatch & CloudwatchLogs


## Conclusion 總結
---

洋洋灑灑列出了記憶中的考試重點以及必考的觀念跟資訊，如果有遺漏再麻煩留言或來信告知，謝謝。

雖說是考試，但在工作上不難發現用到的也都是這些。 有時候一個Bug解不出來也很可能是因為在某個服務的某個關鍵觀念不熟，所以說考試反映出的題目，跟工作上遇到的難題其實是很雷同的。

祝福大家考試都能過關。



## References & Links
---

列出幾個有參考過以及覺得很棒的網站

首推Rick。

[Complete Think - AWS SAA-001 Update (Feb, 2018)](https://rickhw.github.io/2018/02/15/AWS/AWS-Certified-SAA001-Update/)


最神整理，根本Cheat Sheet. 考前閱讀大推！

[Jayendra's Blog - AWS Certified Solutions Architect – Associate SAA-C01 Exam Learning Path](http://jayendrapatil.com/aws-solutions-architect-associate-feb-2018-exam-learning-path/)


網友分享(gjnzsu)

[台部落 - AWS雲服務認證攻略系列（一）AWS Certified Solution Architect Associate 考試經驗分享](https://www.twblogs.net/a/5c2b3aadbd9eee016114880c)





<br />
<br />
<div style="text-align: right;">
2019-12-05 00:23 , David in Taipei</div>

<br />
<br />
<br />
