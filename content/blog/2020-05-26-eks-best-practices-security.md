---
author: David
categories:
- AWS
- Kubernetes
comments: true
date: "2020-06-07"
modified: "2020-06-07"
published: true
title: EKS Best Practices - Security
images: ["https://live.staticflickr.com/65535/49970338736_ffd25b0434_b.jpg"]
---

## Introduction
---
不久前 AWS 出了一本 EKS Best Practices (Mkdocs)，目前只有釋出 Security 的部分，但從 Repository 可以看到未來應該還會有 Cost optimization / Operation / Performance / Reliability 的部分。 由於目前只有 Security 的部分，也就花了幾天把它看完，發現這真的個好東西，那種感覺差不多是你就跟著他跟你說的實踐去做，初學者也不會太偏離軌道。  

根據 AWS 自己說的 AWS 會負責雲的安全 (security of the cloud)，但使用者要負責自己在雲上的安全 (security in the cloud)。 所以我們就來看看我們需要做到哪些事才可以 `盡量` 安全的使用 EKS。由於這份文件說長不長，說短不短，看完還是要一點時間，尤其又是英文，所以我看完後整理了一下重點跟翻譯可以給自己當成筆記。

覺得自己看的滿仔細的，還提交了 5 個 PR，想說以此做個筆記以便日後回來看，寫下來的筆記整理永遠是很好的複習跟加深記憶的方式。 

這裡附上原 Repository 跟原文 Mkdocs Site。

[Github - https://github.com/aws/aws-eks-best-practices](https://github.com/aws/aws-eks-best-practices)  
[Mkdocs Site - https://aws.github.io/aws-eks-best-practices/](https://aws.github.io/aws-eks-best-practices/)

<br />

## 目錄
---

以下內容會分成幾個部分來講，盡量 Follow 原文的結構。

- [EKS & IAM](#EKS-IAM)  
- [EKS IAM Role for ServiceAccounts (IRSA)](#EKS-IRSA)
- [EKS Pod Security](#EKS-POD)
- [EKS Tenant Security](#EKS-TENANT)  
- [EKS Detective Control](#EKS-DETECTIVE)  
- [EKS Network Security](#EKS-Network-Security)  
- [EKS Security Groups](#EKS-SG)  
- [EKS Runtime Security](#EKS-RUNTIME)  
- [EKS Infrastructure(Worker Node) Security](#EKS-HOST)  

<br />

## EKS & IAM {#EKS-IAM}
---

### 認證流程  
首先要先理解 kubectl 怎麼跟 EKS 做認證。

![text](https://live.staticflickr.com/65535/49939885678_bd5fe9294b_b.jpg)

基本上就是你下 `kubectl get pod` -> 會去要 `token` -> 在打到 `EKS API` -> 檢查權限 `aws-auth (configMap)` -> return result

如果想檢查 toekn ，可以從這邊下手。
```bash
aws eks get-token --cluster <cluster_name>
```

這個 Token 預設 TTL 是 15 分鐘，也就是說 15 分鐘後就會過期，如果使用 Kube-Dashboard 的人就要注意一下了，會比較麻煩。

<br />
<br />

### Recommendation  
AWS 建議我們在 IAM 的部分注意以下幾件事

- **盡量不要使用 serviceaccount token 來做認證**  
    因為 SA Token 是 long-lived 的，如果這把 token 被偷了或是你掉在哪了而你不知道，是很危險的事，toekn expiry 的機制是一種保護。 就算是 For CI/CD 也不要用 SA Token，應該要給予一個 IAM Role。  

- **IAM User 有需要賦予 EKS 相關的 Permission 才能 Access Cluster 嗎？**  
    NO, absolutely not. `只需要加在 aws-auth 就可以了`。  

- **如果一次要加很多人到 aws-auth 該怎麼做？**  
    不需要一個一個加，`加一個 Role 到 aws-auth`，然後給 User Assume Role 的權限就可以，相對好管理。  

- **RoleBinding / ClusterROleBinding 最小權限**  
    避免 `["*"]` 在裡面, 盡量一個一個權限給。 `audit2rbac` 是一套會自動幫你依據 K8s audit log of API call 去 grant role & binding 的工具。
    [https://github.com/liggitt/audit2rbac](https://github.com/liggitt/audit2rbac)

- **Make EKS cluster endpoint `PRIVATE`**  
    EKS Endpoint 預設是 public，其實也是安全，因為還有 IAM Auth 的過濾機制，但你還是暴露在 Internet。 AWS 建議盡量設成 `private only`，但這樣理論上你會需要 VPN。  
    如果想要在方便跟安全之間取一個平衡點是可以設定成 public，但請 `務必設定 CIDR Whitelist`。  

- **經常 Audit Access**  
    經常的去看 Cluster 的 Access log (當然是透過工具，不然就工人智慧了)，以下有推薦兩套。  
        [https://github.com/FairwindsOps/rbac-lookup](https://github.com/FairwindsOps/rbac-lookup)  
        [https://github.com/aquasecurity/kubectl-who-can](https://github.com/aquasecurity/kubectl-who-can)

- **除了 IAM，也可以使用 OIDC 授權 Github 使用 OAUTH 存取 EKS (或是 AWS SSO)**  
    不過一般情況下還是推薦使用 IAM （權限比較可以集中單一管理，不易發散）。 不過如果有 Kubernetes-Dashboard / Kibana 這一類的需求，也可以用 OIDC 做 OAuth 來做證認。  
    - Kubernetes Dashboard SSO with OIDC [https://thenewstack.io/single-sign-on-for-kubernetes-dashboard-experience/](https://thenewstack.io/single-sign-on-for-kubernetes-dashboard-experience/)  
    - EKS with Github  [https://aws.amazon.com/blogs/opensource/authenticating-eks-github-credentials-teleport/](https://aws.amazon.com/blogs/opensource/authenticating-eks-github-credentials-teleport/)  
    - OIDC Proxy [https://aws.amazon.com/blogs/opensource/consistent-oidc-authentication-across-multiple-eks-clusters-using-kube-oidc-proxy/](https://aws.amazon.com/blogs/opensource/consistent-oidc-authentication-across-multiple-eks-clusters-using-kube-oidc-proxy/)


<br />

## EKS IAM Role for ServiceAccounts (IRSA) {#EKS-IRSA}
---

### IRSA 介紹  
這個主題特別拉出來講，因為他是 Kubernetes 原生所沒有的。 IRSA 簡單講就是  
`IAM Role` ↔ `IRSA` ↔ `ServiceAccount`  

整個跟認證結合起來就會是  
`Pod` → `AWS OIDC` → `EKS sign OIDC token` → `AWS API` → `AssumeRole` → `IAM Role`

```md
就是 Pod 會經過 OIDC token 去交換 IAM Role 來存取 AWS Services (e.g. S3)
```

> 上面講到 Token 不過期很危險，這邊的 OIDC Token 會由 `kubelet` 去替換。
> 替換的依據是當這個 Token 達到 80% TTL 或者是超過 24HR。

<br />
<br />

### Recommendations
- **Disable 不需要的 ServiceAccount 權限**  
    如果你的 applicaion 不需要使用到 K8S API，可以設定  
    ```
    kubectl patch serviceaccount default -p $'automountServiceAccountToken: false'
    ```

- **每個 Application 都有自己專用的 ServiceAccount 比較好**  
    就跟 ECS 每個 task 都有自己的 task role 一樣。 權限最小化。

- **Worker 的 Instance Profile 只留必要的**  
    當我們使用 IRSA 的時候, Pod 就不再使用 Instance Profile 了，所以大可將它權限鎖住 (block all)  
    我們要讓所有的 Pod 都強制使用 IRSA (也就是走 Service account → IAM Role 的授權方式)

- **使用非 Root 的用戶來啟動 Containers (`runAsUser` / `runAsGroup`)**  
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: security-context-demo
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
      containers:
      - name: sec-ctx-demo
        image: busybox
        command: [ "sh", "-c", "sleep 1h" ]
    ```

- **IRSA 的 Role Trusted Relationship 盡量綁定 service account**  
    這是指 trusted relationship 那邊的設定，建完 Role 之後要手動去改 （建立的時候沒有這選項） 
    範例中就是將這個 Role 限制在 `deault namespace` 的 `exteranl-dns service account`  
    ```yaml
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Federated": "arn:aws:iam::xx:oidc-provider/oidc.eks.ap-northeast-1.amazonaws.com/id/xxx"
          },
          "Action": "sts:AssumeRoleWithWebIdentity",
          "Condition": {
            "StringEquals": {
              "oidc.eks.ap-northeast-1.amazonaws.com/id/xx:sub": "system:serviceaccount:default:external-dns"
            }
          }
        }
      ]
    }
    ```

<br />
<br />

### IRSA 的限制
儘管 IRSA 很美好，可以幫你跟 AWS 溝通，但他也有限制，就是 Application 使用的 `AWS SDK 版本問題`。  
你必須要確保你的 SDK 版本可以支援 IRSA (太舊不行)。 如果不支援，可以使用 kube2iam 或是 kiam 來替代。

<br />

## EKS Pod Security {#EKS-POD}
---
### Pod Security Policy (PSP)
PSP 是用來控制 Pod 權限的，而 EKS 建立時會自動建立一個 `eks.privileged` PSP。  

協助管理 PSP 的工具 - [kube-psp-advisor](https://github.com/sysdiglabs/kube-psp-advisor)  

<br />

### Recommendations
- **Pod 只要設定 `privileged` 就可以拿到很多 Node 上 root 的權限，需要盡量避免。**  

- **Container 避免使用 root 啟動，也不要給 sudo 權限**  
    有 root 權限的 Pod，相對有機會被進行提權攻擊，會建議使用者盡量使用 PSP 去強制規範。  
    可以考慮進行以下步驟來加強安全性：  
    - 移除 `shell`  
    - 指定 non-root 的 user 在 Dockerfile  
    - Podspec.securityContext 內可以指定運行的 User (`runAsUser` / `runAsGroup`)  

- **不要 Docker in Docker**  
    這樣一來 Node 就無法幫你管理 in Docker 的這個 Container，會很危險，等於你只受到上一階層的 Container 直接管理而已。  

- **避免使用`hostPath`**  
    在 Serverless 的角度，我也覺得盡量不要存東西在 Host 上，可以考慮 EFS 或直接使用 Fargate。 
    AWS 在這邊是說真的要用的話記得要注意權限 (readonly 該給的要給)。  
    另外也可以在 PSP 先做好限制（如下面範例）。  
    ```yaml
    allowedHostPaths:
    # This allows "/foo", "/foo/", "/foo/bar" etc., but
    # disallows "/fool", "/etc/foo" etc.
    # "/foo/../" is never valid.
    - pathPrefix: "/foo"
        readOnly: true # only allow read-only mounts
    ```

<br />

### Quality of Service (QoS)

Kubernetes 使用 QoS 去排列優先度，決定什麼時候要殺掉哪些 Pod。  
QoS 很重要，不僅是資源利用跟效率的問題，設定上的失誤也會造成 Pod 一直 evicted。

|  Name        | Priority  |  Condition                                                   |  Result                                        |
|--------------|-----------|--------------------------------------------------------------|------------------------------------------------|
|  Guaranteed  |  `highest`  |  limit = request ≠ 0  <br />(只設 limit, 那 request 會自動等於 limit)    | 除非超過 memory limit 不然不會被殺        |
|  Burstable   |  `medium`   |  limit ≠ request ≠ 0                                         | 超過 request memory 後如果 Node 資源不夠就會被殺    |
|  Best Effort |  `lowest`   |  limit & request = null                                      | 資源不夠最早被殺                                  |


製作了一張表來看 QoS 的分級，由此可見如果 `request` 跟 `limit` 都不設定，那就會相對比較危險 (Pod 之間在資源不足的時候會競爭)。
另外設定 QoS 也可以間接的防止 DOS Attack，避免 Node 上的資源一直被某個 Pod 過度調度造成其他 Pod 都被 kubelet 驅逐 (evicted)。  

<br >

**`Request` & `Limit` 有一些特點：**  
- 對每個服務設定不同的 QoS 是必須的。  

- 盡量透過 Namespace `Quota` 和 `Limit Range` 去做限制。
    - Namespace Quota 可以決定要給裡面的 Pod 多少資源（Total amount of resources）。
    - Limit Range 可以決定 Namespace 裡資源的 min/max。

- CPU 是可以超額調度(oversubscribed) 的，Memory 不行。  
    e.g. 多容器間 Memory 是不能共享的，但 CPU 可以。

- 當 Node 上存在內存壓力時 (Memory pressure) ，超過 request memory 但未達 limit 的 Pod 可能會被終止。

- Memory Limit 對應到的是 `cgroup` 的 `memory.limit_in_bytes`，所以超過的 Pod 會直接 OOM Killed。 CPU Limit 如果超過則是會受到節流(throttled)。

- Memory Request 則不受到 cgroup 影響，但如上面所說如果 Node 有壓力還是會被終止的。

<br />

## EKS Tenant Security {#EKS-TENANT}
---
Kubernetes 在多租戶 (Multiple Tenant) 的應用下，要怎麼去做到對每個租戶的`隔離`，是很重要的一件事。 所謂`多租戶`，就是指一組應用程式所使用的計算，網絡，存儲等資源組成的工作負載集合。也就是說會有需要不同資源的不同應用程式跑在同一個 Kubernetes Cluster。 而如何適當隔離不同應用，避免惡意的租戶去對其他租戶做攻擊，並保證租戶間可以分配到各自合適的資源，就是這個 section 要探討的。  

Kubernetes Control Panel 本身是單租戶的形式 (一個 control panel 給所有租戶共用)，但在 Kubernetes 集群中我們可以透過一些限制達到相對安全的多租戶應用的形式，以便更有效率地利用資源。  

我覺得這邊解釋應該是說：安全性理想狀況下，一個 Kubernetes 環境中只運行一個應用程式，這是`單租戶`。 但我們透過對 Namespace 隔離等等的手段，使得我們可以讓多個應用程式跑在同一個 cluster，並確保彼此之間不會互相影響以達成運行多個應用在同一個 Kubernetes 環境的需求，這是`多租戶`。  

單/多租戶可以想成是房子的住戶形式。一棟透天厝只住一戶就會相對安全，也不用搶資源（單租戶）。 如果是一棟公寓裡面住了很多戶，要怎麼安排每位住戶的資源分配跟每一戶的隔離性，就會是問題了（多租戶）。（總不能讓 A 隨便穿牆可以跑到 B 家，或是 A 要洗澡 B 就不能洗碗這類的狀況發生） 

我自己在日常實踐中也多是 Soft 多租戶，切不同的 Namespace 並用 RBAC / Limits 來做限制給不同的應用程式或是 Team 做使用。  

<br />

### Soft Multi-Tenant
Soft 就是透過邏輯層隔離的方式來達到這個目的。  
- RBAC
- Quotas
- Limit ranges
- Different Namespace

以上這些都是基本的，但卻不能阻止 Pod 共享一個 Instance。 為什麼不能共享 Instance? 因為機器裡有我們的 secret / configmap / volume 資訊，萬一掉了一台，可能就一次掉了好幾個應用程式的機密資訊。但這一切端看使用者對於隔離的 `需要性`，要做到強度多高的隔離，完全是依照本身的需求跟限制去執行的，沒有標準答案。

如果需要更強大的隔離，可以考慮以下方法：
- Node Selector
- Anti-Affinity Rules
- Taints / Tolerations

這些方式可以讓 Pod 只部署在特定的機器上，以達到底層機器隔離的效果。

Kiosk 是一款可以協助我們導入 soft multi-tenancy 的工具  [https://github.com/kiosk-sh/kiosk](https://github.com/kiosk-sh/kiosk)

<br />

### Hard Multi-Tenant
Hard 就是很硬的把每個 deployment 放在不同的 cluster，當然這是最有效的資源隔離方式，也是所謂 0 信任。  
但我們必須要考慮到：
- 會變很貴
- 導致資源利用不足
- 很難管理

<br />

### Conclusion
```
所以一般情況下還是考量 Soft multi-tenancy 就可以了。
```

考量到企業多半是採取 `半信任(semi-trusted)`，人是不可信的情況下，必須得要做適當的隔離。一般來說系統管理員負責切分 Namespace ( by Application) ，並賦予每個 Team 最小所需的權限（只給需要的Namespace）並搭配 PSP 使用。

而考量到 Soft/Hard 都各有優缺點的情況下，Multi-Tenancy Special Interest Group (SIG) 正在改善 Soft/Hard 的不足。
正朝著以下方向邁進：  
- Namespace 加入繼承的概念 (subnamespace)
- Virtual Cluster (不同 service 可以建立個別的 Nodes，Node 裡包含 API / controller / scheduler，就是所謂`Kubernetes on Kubernetes`)

<br />

## EKS Detective Control {#EKS-DETECTIVE}
---

### Auditing and Logging
- 稽核 Log 通常可以分析及發現很多問題
- 建議啟用 AuditLog on EKS, 就會送到 CloudwatchLogs (但要收費)

### 利用 Audit Log
Kubernetes audit log 會包含兩個 annotations  
- [`authorization.k8s.io/decision`](http://authorization.k8s.io/decision) → 指出 request 是否已授權  
- [`authorization.k8s.io/reason`](http://authorization.k8s.io/reason) → 授權原因  
- 善用兩個 annotations 去驗證 API Call 是否合法  

### 建立 CloudWatch Alarm
- 建立 CW Alarm 偵測 401 / 403 Forbidden 的 API Call 數量，並在有 403/401 的時候報警。
- 然後使用 host / sourceIP / k8s_user.username 去找出是哪個地方的漏洞。
- 或是使用 CloudwatchLog Insight 去監控
- CloudTrail Log 也需要監控 (因為必須要知道 IRSA 的使用 e.g. 是誰從Pod打了 AWS API Call 到 S3)。

### Log 的管理
Log 一直長大，管理跟過濾變得越來越麻煩跟昂貴。可以考慮用 `Falco` (CNCF) (Runtime security tool) 分析 audit log 分析異常現象 [https://github.com/falcosecurity/falco](https://github.com/falcosecurity/falco)  

#### - Falco
- 用 Helm 安裝在 EKS 裡就可以，還有包含預設的 Rules
- 串接 Lambda 發送 Alarm
- 官方教程： [https://aws.amazon.com/cn/blogs/china/securing-amazon-eks-lambda-falco/](https://aws.amazon.com/cn/blogs/china/securing-amazon-eks-lambda-falco/)
- 還可以用 ekscloudwatch 將 log 從 CWLogs 送到 Falco 去做分析 [https://github.com/sysdiglabs/ekscloudwatch](https://github.com/sysdiglabs/ekscloudwatch)
- 另一個選項是 S3 + SegaMaker

<br />

## EKS Network Security {#EKS-Network-Security}
---
Pod 之間預設是可以交流的，這並不安全。 所以 Kuberentes Network 提供我們在 Pod 之間網路做限制的方法 (東西向流量), 或是 Pod 跟外部服務溝通的網路限制方法。
```text
簡單來說就是 Kubernetes 自己的 Security Group，只是有些包含到 OSI Layer 3 - 4。
```
常見的有以下幾種：
- Calico
- Tigera
- Cilium


### Encryption in Transit (傳輸時加密)
- Nitro Instance (C5n, G4, I3en, M5dn, M5n, P3dn, R5dn, R5n) 是預設傳輸加密的。
- CNI 的選擇 (使用 WeaveNet 也會自動加密)
- Service Mesh 也可以做到
    - AWS App Mesh → SSL (使用 ACM Private Certificate )
    - Linkerd → SSL
    - Istio → SSL
- 使用 Ingress Controller with HTTPS
    - ALB / NLB / CLB
    - 如果你要 Pod 到 Pod 走 HTTPS 流量加密，是很吃 CPU 資源的，因為你會需要在 Pod 做加解密並建立 handshake，而這很浪費資源，因為 Pod 資源理論上是要給應用程式做使用的。要盡量在 Ingress (最上層）去做完這件事就可以了。


### Recommendations
- 建立 K8S Network Policy 或 `Calico` 裡限制 inbound/outbound 流量。
    - Kubernets Network Policy
    - Calico Global Network Policy
- 盡量讓 Pod 之間使用 DNS 溝通，並建立允許 DNS (port: 53) 流量的規則。
- 只允許必要的流量。
- NLB / ALB 要加上 SSL Certificate。

<br />

## EKS Security Groups {#EKS-SG}
---
官方建議：[https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html)

Node / Control Panel :   
    - **10250** (kubelet needs)  
    - **443** (API needs) 
    - 這兩個 port 一定要開。

當我們建立 cluster 的時候, SG 會自動幫我們建立。
- 為簡單起見，建議 `將 Control Panel 的 SG 直接加到 Worker SG` 確保溝通上沒有問題。
- 如果需要跟外部服務溝通，可以使用 Network Policy Engine (e.g. Cilium 使用 DNS 去溝通) or (Calico 使用 network policy 去幫你建立 SG)
- 或是使用 Service Mesh (Istio, app mesh) 也可以控制 egress gateway
- `不要在 Control Panel & Managed Node Group 上面掛同一個 SG`，會引發無法刪除 Node Group 的 Bug。 [https://docs.aws.amazon.com/eks/latest/APIReference/API_Issue.html](https://docs.aws.amazon.com/eks/latest/APIReference/API_Issue.html)


> 建議使用 Network Policy 去控制的原因是因為我們不會在一台 Instance 上只跑一個 Pod ，所以使用控制 VM Level 的 SG 去控制 Pod 會有點粗糙，畢竟我們還無法把 SG 套用在個別的 Pod 身上。

<br />

## EKS Runtime Security {#EKS-RUNTIME}
---

> 目的是為了防止在 container 內的惡意攻擊，防止被從 container 內做 system call 打到底層的 host kernel。

為了達到目的，我們應該限縮 application 利用到 system call 的範圍。
而我們可以利用 `seccomp (secure cimputing)` 來協助。 seccomp 是 Linux 的一個工具，可以限制 process 只能呼叫特定的 system call。  
- syscall2seccomp  [https://github.com/antitree/syscall2seccomp](https://github.com/antitree/syscall2seccomp)

跟 SELinux 不一樣，`seccomp` 不是要獨立容器，他是要保護 host kernel 不要被未授權的執行 syscall。他會只允許在 whitelist 內的 syscall。而 Docker 有預設的 seccomp profile 可以去使用並符合大部分的需求。  
- seccomp 在 kubelet 上是 alpha feature, 可以使用 `--seccomp-profile-root` 去啟用。
- 使用 3rd 方案保護 runtime
    - Apparmor
    - seccomp profile

補充資料[Docker Seccomp Profile 配置]：
[https://blog.csdn.net/kikajack/article/details/79596843](https://blog.csdn.net/kikajack/article/details/79596843)

<br />

## EKS Infrastructure(Worker Node) Security {#EKS-HOST}
---
- **定期汰換 Node**
    - 升級時不要 in-place，`直接把 Node 都換成新的機器` (Managed Node Group 推薦這樣做)。
    - Node Group 整組換掉自動化步驟，以免人為失誤（這很常有失誤。
        - 建立新的 Group
        - Drain containers
        - Terminate Old Instances
    - Fargate 會自動在每次啟動都使用新的，這也是一種方法（原本就是 Fargate 的優點。
        - 但會常常被 reschedule，建議都使用 deployment / statefulset。

- **定期檢查 cluster 是否合乎安全規範**
    - kube-bench  [https://github.com/aquasecurity/kube-bench#running-in-an-eks-cluster](https://github.com/aquasecurity/kube-bench#running-in-an-eks-cluster)
    - kube-bench 目前 EKS AMI 會有假陽性問題，正在 issue 裡等待修復

- **最小化 Node 權限**
    - 使用 SSM Session Manager 取代 SSH (反正 For Production 理論上也不會常進去)。
    - 但 SSM Session Manager 目前有問題，你會需要在 Host 上安裝 SSM-Agent，那如果你一開始就拔掉 SSH，又要怎麼去安裝？ (`雞生蛋、蛋生雞問題`)
        - 目前可以使用一個 Privileged 的 DaemonSet 去跑 Shell Script 去安裝，但跑完就要把這個 DaemonSet 刪掉
        - 等到官方正式支援就可以不用幹這件事了 QQ
        - The DaemonSet 參考安裝：  [https://github.com/jicowan/ssm-agent-daemonset](https://github.com/jicowan/ssm-agent-daemonset)
        - 原理是利用 `nsenter` 進到別的 namespace 去提權安裝 （hack自己)

- **把 Worker Nodes 放在 Private Subnets**
    - 在 4/22, 2020 之前建立的 NodeGroup 都預設會給 Node Pulic IP, 如果是這樣你可以透過 SG 限制。
    - 之後的都會根據 Subnet 來決定要不要給 Public IP，所以把 Worker Node 放在 Private Subnet 就 ok 了。

- **使用 Amazon Inspector 來檢驗是否有漏洞**  
[https://docs.aws.amazon.com/zh_tw/inspector/latest/userguide/inspector_introduction.html](https://docs.aws.amazon.com/zh_tw/inspector/latest/userguide/inspector_introduction.html)  
Amazon Inspector 會測試 Amazon EC2 instances 的網路存取能力，以及這些執行個體上執行的應用程式安全性狀態。Amazon Inspector 會評估應用程式是否有漏洞與弱點，以及是否偏離最佳實務。執行評估後，會產生安全調查結果的詳細清單，並依嚴重程度排列。

    - 會需要安裝一個 deployment 去監控機器上的活動。
    - 如果你用 NodeGroup 的話目前不支援你使用自己的 AMI，所以你會需要在 NodeGroup 幫你啟動 (bootstrapped) 前先將 Agent 安裝好。
    - Fargate 不能跑 Inspector。

- **使用優化的 OS (Container 專用)**
    - RancherOS
    - Bottlerocket 🔥

- **其他替代做法： 使用 SELinux**  
[SELinux, Kubernetes RBAC, and Shipping Security Policies for On-prem Applications](https://platform9.com/blog/selinux-kubernetes-rbac-and-shipping-security-policies-for-on-prem-applications/)



<br />
<br />
