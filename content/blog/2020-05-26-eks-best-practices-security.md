---
author: David
categories:
- AWS
- Kubernetes
comments: true
date: "2020-06-04T00:00:00Z"
modified: "2020-06-04"
published: true
title: EKS Best Practices - Security
images: ["https://live.staticflickr.com/65535/49970338736_ffd25b0434_b.jpg"]
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

<br />

## 目錄
---

以下內容會分成幾個部分來講，盡量 Follow 原文的結構。

- [EKS & IAM](#EKS-IAM)  
- [EKS IAM Role for ServiceAccounts (IRSA)](#EKS-IRSA)
- [EKS Pod Security](#EKS-POD)
- EKS Tenant Security  
- EKS Detective Control  
- EKS Network Security  
- EKS Runtime Security  
- EKS Infrastructure(Worker Node) Security  

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
    不過當然還是推薦使用 IAM。 綁其他 3rd OAuth 如果不是有特定原因真的不太建議。  
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