---
author: David
categories:
- AWS
- Kubernetes
comments: true
date: "2020-05-26T00:00:00Z"
modified: "2020-05-26"
published: false
title: EKS Best Practices - Security
---

## Introduction
---
AWS 出了一本 EKS Best Practices (Mkdocs)，目前只有釋出 Security 的部分，從 Repository 可以看到未來應該還會有 Cost optimization / Operation / Performance / Reliability 的部分。 

由於目前只有 Security 的部分，我也就花了幾天把它看完，發現這真的個好東西，那種感覺差不多是你就跟著他跟你說的實踐去做，初學者也不會太偏離軌道。

雖然其實大部分都滿基本的，但我還是很推薦都是高手的大家去看看，搞不好有意外的收穫。 

根據 AWS 自己說的 AWS 會負責雲的安全 (security of the cloud)，但使用者要負責自己在雲上的安全 (security in the cloud)。 所以我們就來看看我們需要做到哪些事才可以 `盡量` 安全的使用 EKS。

由於這份文件說長不長，說短不短，看完還是要一點時間，尤其又是英文，所以我看完後整理了一下重點跟翻譯可以給有需要的朋友作參考，或是當懶人包看也可以。

我看的滿仔細的，還提交了 5 個 PR，應該是盡量可以捕捉到作者所要提的重點。 

這裡附上原 Repository 跟原文 Mkdocs Site。

[Github - https://github.com/aws/aws-eks-best-practices](https://github.com/aws/aws-eks-best-practices)  
[Mkdocs Site - https://aws.github.io/aws-eks-best-practices/](https://aws.github.io/aws-eks-best-practices/)



## 目錄
---

以下內容會分成幾個部分來講，盡量 Follow 原文的結構。

- [EKS & IAM](#EKS-&-IAM)  
- EKS Pod Security  
- EKS Tenant Security  
- EKS Detective Control  
- EKS Network Security  
- EKS Runtime Security  
- EKS Infrastructure(Worker Node) Security  



## EKS & IAM
---

首先要先理解 kubectl 怎麼跟 EKS 做認證。

![text](https://live.staticflickr.com/65535/49939885678_bd5fe9294b_b.jpg)