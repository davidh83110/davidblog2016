---
author: David
categories:
- AWS
- Kubernetes
- Terraform
comments: true
date: "2020-03-30T00:00:00Z"
modified: "2020-03-31"
published: true
title: EKS Building & Troubleshooting
images: ["https://live.staticflickr.com/65535/49715657891_23558e7f5d_k.jpg"]
url: /aws/kubernetes/系統建置/terraform/2020/03/30/eks-install.html
---

This is the full map of EKS and we'll go through how to install EKS and plugins in this post, as well as the troubles I got. Let's go!


## Quick Start an EKS with ADD-ONs
---

```bash
git clone https://github.com/davidh83110/eks-quickstart
cd terraform && make plan && make apply
cd ansible-playbooks
ansible-playbook ./
```

You will get an `EKS cluster` and `Node Group`, also has 
`metrics-server` / `k8s-dashboard` / `ALB-Ingress controller` / `Cluster-Autoscaler (CA)` / `Prometheus` / `Grafana` / `Weave`.  

<br />

## Pre-Create Cluster
---

**What EKS Cluster needs:**  
- `eksServiceRole`
- Subnets (Including Public and Private Subnets)
- Additional Security Groups
- Endpoint Access 
- Secrets Encryption
<br />

最容易有疑問的大概是上述這幾個參數了，一一詳細記錄下來，其他沒有寫的應該都能很輕易的選擇。

- **eksServiceRole**  
這個有用過ECS應該都不陌生，EKS也需要一個類似的東西。  
需要兩個Policies. `AmazonEKSServicePolicy` & `AmazonEKSClusterPolicy`


- **Subnets**  
Cluster的Subnets務必要選擇Public & Private Subnets, **不能只選Private**.  
所謂Private就是要給你的Node用的; Public就是要給ALB-Ingress用的。


- **Additional Security Group**  
這個是用來控制誰可以存取代管的MASTER.  
EKS會在創建的時候，自動幫你的Cluster掛上代管的Security Group，當你在啟動新的ALB-Ingress/NodeGroup的時候，這個代管的SG會自行變動Rules.  
Additional就是指除了這個代管的之外，你可以選擇其他你自由控制的SG附加上去。
`BUT, 一但建立之後就不能改變附加的SG了`


- **Endpoint Access**  
這個是控制誰可以使用Kubernetes API, 也就是誰可以用kubectl去控制這個cluster.  
For instance: 如果你希望要開VPN才可以使用，你就只需要勾選`Private Access`.  
但如果希望可以不要這麼麻煩的還要開VPN, 可以同時勾選`Public Access` & `Private Access`, 並指定`Source to public access endpoint`.

> 如果勾選了Public Access, 一定要給Source IP.  
> 因為如果不給的會會套用預設值，也就是 0.0.0.0/0. 非常危險，你的cluster就裸奔了。


- **Secrets Encryption**  
這是可以讓你綁定KMS Key去加密Kubernetes Secrets.  
`因為Kubernetes Secrets眾所皆知就是做base64 ENCODE 而已`  
所謂Encode就是可以直接Decode (base64 -d), 是編碼的一種，`並不算是加密!`  
    
    之前常見的做法會是使用Hashicorp的 `Vault`, 然後掛Sidecar在Runtime的時候去取.   
    現在EKS提供和KMS的整合，讓我們在存進去的時候他就用KMS再做一層加密了，  
    `只是我還不知道怎麼去驗證AWS確實有幫我們做加密, 這也是滿tricky的地方`。
    
    
- **AWS Auth**  
之前會需要apply `aws-auth.yaml` 這個ConfigMap, 但現在(1.15)似乎EKS已經自己幫我們Apply了。  
如果要修改或新增 IAM User/Roles, 可以再去編輯這個 ConfigMap 就好。  

<br />

## What else do I need to do after create an EKS cluster ?
---

先進到EKS Console, 來看看有什麼相關資訊。 以下列舉幾個比較可能有疑問的。

- **Platform version**  
AWS自己給的eks版本號碼，基本上不太需要理他。


- **API server endpoint & Certificate authority**  
你的Master API Endpoint跟Cert. 不過透過aws cli去做授權的話，也不太需要管這個。


- **OpenID Connect provider URL**  
OIDC. 這是讓EKS可以結合IAM使用做授權管理的關鍵，會需要拿這個ID去IAM OIDC那邊建立一個新的`Identity providers`.  
後面會詳細講到如何建立。 主要是在建立 ALB-Ingress Controller的時候會需要去做建立。


- **Node Group**  
在之前要建立EKS的Nodes, 會需要自己去建立Auto Scaling Group(ASG)並連結到EKS, 還要打一些Tags.  
現在透過Node Group可以半代管ASG, 他會自動根據你給的資訊去建立ASG, 並自動連結到EKS. (雖然常常有些問題)。  
後面也會詳細講到建立過程。


- **Fargate Profiles**  
Fargate with EKS是沒有Node(worker)的概念的。就像ECS沒有ECS Instance一樣。  
所以這邊會去指定Fargate所在的Subnets和Role，提供給Pod在上面跑的資訊而已。  

<br />

> 比較值得說的是Fargate的使用模式  
> 
> Fargate with EKS是有限制的，請以Serverless的角度來思考。  
>
> Fargate features with EKS:
> - No Daemonset
> - No statefulset that require PV or File Systems (EFS also no supported)
> - Maximum 4vCPU and 30G Memory per pod
> - No CLB/NLB (cuz no NodePort in Fargate)

[More Information: AWS Blog - Fargate with EKS](https://aws.amazon.com/blogs/aws/amazon-eks-on-aws-fargate-now-generally-available/)

<br />

## 第一件要做的事 - TAGGING
---

由於EKS繼承了Kops的模式，大量使用Tags來控制EKS的相關資源，也很容易被忽略，一定要做好這部分的控制。

`kubernetes.io/cluster/${CLUSTER_NAME} ` 是最廣泛使用的Tag, 會需要Tag在Subnets, Instances上。 可以Assign的值有兩個 - `owned` OR `shared`。 

一般來說在Subnets都會給`owned`, 因為會需要與其他EKS Cluster共用, 除非你希望這個Subnet是給某個cluster專用。
`shared` 顧名思義就是共用subnet。  


`elb` OR `internal-elb` 這個分別要Tag在Public & Private Subnet上面，讓EKS知道他們是什麼類型的子網，
這樣ALB-Ingress才能根據Tag放到正確的子網。


需要被Tag的有以下資源：

- **Public Subnets**  

    ```yaml
    kubernetes.io/cluster/${CLUSTER_NAME} = shared
    
    kubernetes.io/role/elb = 1
    ```

- **Private Subnets**  

    ```yaml
    kubernetes.io/cluster/${CLUSTER_NAME} = shared
    
    kubernetes.io/role/internal-elb = 1
    ```

- **Security Group / Instances / NodeGroup**  
    這些理論上EKS會自動去檢查，然後幫我們tag上去，不要手賤去刪都行。
    而我相信EKS也是往這個方向努力，之後會自己幫我們檢查TAG並自動做掉這部分，讓我們拭目以待。

<br />

## Create Nodes
---

建立Nodes有兩種方式  
- 自行建立Auto Scaling Group(ASG) & ASG Configuration & Register to EKS
- 使用EKS Node Group


過去通常都要自行註冊與建立ASG, 是有點麻煩。 現在可以使用Node Group來幫我們`半代管`這個Nodes。  

為什麼是`半代管`？

```
當Node Group出現註冊問題或是有錯誤時，你還是得要進去這台機器手動排除錯誤。
```

<br />

#### **建立Node Group的注意事項:**  


- **VPC** (務必檢查，很容易忽略)
    - `MUST ENABLE enableDnsHostnames, enableDnsSupport`
    - [Check Documentation Here](https://docs.aws.amazon.com/zh_tw/eks/latest/userguide/cluster-endpoint.html)

- **Subnets**  
    選擇Private Subnets

- **eksNodeRole**  
    Attach 3 Policies:
    - AmazonEC2ContainerRegistryReadOnly
    - AmazonEKS_CNI_Policy
    - AmazonEKSWorkerNodePolicy

- **Remote Access Security Group**  
    要注意的是這裡指的是 `允許Remote Access` 的SG, 不是要掛在Instance上面的SG。

- **Labels**  
    這邊指的是 `Node Group的TAG, 不是Instance的TAG`
    
<br />

## Kubectl
---

做完以上那些大概要花10-20min去跑，跑完都是 `Active` 的狀態差不多就完成了。  
接下來我們實際 Access 這個 cluster 看看。  


首先，需要安裝 `kubectl`. (自行安裝)  
然後需要建立相對應的 `kubeconfig`, EKS 會使用 `aws-iam-authenticator` 來幫我們完成這部分。


過程如下：  

```bash
➜  ~ aws eks --region ${REGION} update-kubeconfig --name ${CLUSTER_NAME}                                                                                                                                                           <aws:dev>
Updated context arn:aws:eks:ap-northeast-1:34323434:cluster/development in /Users/davidhsu/.kube/config
➜  ~                                                                                                                                                                                                                                <aws:dev>
➜  ~ kubectl get nodes                                                                                                                                                                                                              <aws:dev>
NAME                                              STATUS   ROLES    AGE    VERSION
ip-10-10-11-141.ap-northeast-1.compute.internal   Ready    <none>   125m   v1.15.10-eks-bac369
ip-10-10-12-203.ap-northeast-1.compute.internal   Ready    <none>   126m   v1.15.10-eks-bac369
➜  ~
```

[Reference Official Documentation - EKS kubeconfig](https://docs.aws.amazon.com/zh_tw/eks/latest/userguide/create-kubeconfig.html)


到這裡這個 EKS Cluster 就算是完成囉，但是還需要一些額外設定跟 Add-ons 的安裝。

<br />

## ADD-ONs of EKS
---

這邊詳列要裝的 Add-ons 並附上官方的資料來源（官方怎不幫我裝一裝就好？）  


- **Metrics-Server**  
    收集 kubernetes cluster 內各項指標，以便提供給 Prometheus/K8s Dashboard 收集使用，或做圖表的呈現。  
    [安裝 Kubernetes 指標伺服器](https://docs.aws.amazon.com/zh_tw/eks/latest/userguide/metrics-server.html)


- **Kubernetes Dashboard**  
    Kubernetes Web UI, 提供視覺化的操作以及資源的監控，必須要先建立 `eks-admin` 這個 ServiceAccount。  
    [部署 Kubernetes Web UI (儀表板)](https://docs.aws.amazon.com/zh_tw/eks/latest/userguide/dashboard-tutorial.html)


- **Cluster Autoscaler (CA)**  
    用來自動擴展 Node 數量的套件，當有資源需要 Node 而 Node 不夠的時候會自動加開新的 Nodes。 

    這和 `ASG` 是連動的，會自動調整 Desired & Minimum Count，但是受到 Maximum 的限制，也就是說自動成長到定義的 Maximum 之後就不會再長了。  
    [EKS - Cluster Autoscaler](https://docs.aws.amazon.com/zh_tw/eks/latest/userguide/cluster-autoscaler.html)


- **ALB Ingress Controller**  
    與ALB連接的Ingress Controller，可以直接在 ingress yaml 裡直接定義 ALB 的參數。

  
    比較可怕的是如果自定義 Security Group 的 ID ，會造成 Master SG 沒有自動加入這個 ALB 的 SG 到 Inbound，就會一直是 unhealthy 的狀態，可以透過指定 inbound rule 的方式來避免這個 Bug。  

    ```yaml
    alb.ingress.kubernetes.io/security-group-inbound-cidrs: 61.67.11.22/32
    ```  
    [Amazon EKS 上的 ALB 傳入控制器](https://docs.aws.amazon.com/zh_tw/eks/latest/userguide/alb-ingress.html)  
    [ALB Ingress Security Group's Issue](https://github.com/kubernetes-sigs/aws-alb-ingress-controller/issues/691)


- **Prometheus**  
    [控制平面指標與 Prometheus](https://docs.aws.amazon.com/zh_tw/eks/latest/userguide/prometheus.html)


- **Grafana**  
    [EKS Workshop - DEPLOY GRAFANA](https://eksworkshop.com/intermediate/240_monitoring/deploy-grafana/)

<br />

## Troubleshooting
---

### **Q: Node cannot join to cluster? ( `cni config uninitialized` )**  
A: Please make sure subnets are well-tagging and all selected in EKS cluster.  
If you're using NodeGroup then it won't has any problem on your instances basically, 
so what you need to check is the resources about VPC and Subnets.  
- Subnets tags
    - `kubernetes.io/role/elb` = `1` (For Public Subnets (Ingress Subnets))  
    - `kubernetes.io/role/internal-elb` = `1` (For Private Subnets (Nodes Subnets))  
    - `kubernetes.io/cluster/${CLUSTER_NAME} ` = `shared` (ALL Subnets including for ingress and nodes)  
- You must selected `PUBLIC & PRIVATE` subnets in EKS control plane.
- VPC DNS Hostname `enabled`
- VPC DNS Support `enabled`

Also, you can use `journalctl -u kubelet` on nodes to check the logs.

<br />

### **Q: error: You must be logged in to the server (Unauthorized)**  
A: Your identity might be wrong. Please check the current identity by the following command.

```bash
- $ aws sts get-caller-identity
{
    "Account": "000012345",
    "UserId": "dawdwwfwefewfefwef",
    "Arn": "arn:aws:iam::000012345:user/david"
}
```  
Then rerun the kubeconfig setup.  
```bash
- $ aws eks --region ap-northeast-1 update-kubeconfig --name test-cluster                                                                                                                                                           <aws:dev>
Updated context arn:aws:eks:ap-northeast-1:000012345:cluster/test-cluster in /Users/davidhsu/.kube/config
```

<br />

### **Q: Security Group on EKS Control Plane?**  
A: EKS Control plane has 2 kinds of subnets.  
- EKS Master Managed SG  
- Additional SG  

When we are creating some AWS resources such as ALB-Ingress's SG, 
it will be automatically added to Master managed SG, so we don't have to worry about it.

Additional SG is for other source you trust, you can add them to the SG so that they can access the EKS Master as well.

<br />

### **Q: API server endpoint access?**  
A: If you are using `private access` for the connectivity between nodes and master, then you must `enable VPC DNS hostname and support`. Otherwise, it will still go with public route. 

<br />

### **Q: Fargate with EKS**  
A: `Fargate is a serverless service.`

It doesn't support `daemonset` / `statefulset` / `EFS`.  
Only `10G` volume can be used per Fargate; and the maximum of vCPU and Memory are 4vCPU and 30G RAM.

To use Fargate on EKS, you need to create `Fargate Profile` first.

<br />

### **Q: Can I specify Security Group for ALB-Ingress?** 
A: Yes. But the EKS Managed SG might not be changed and it means you have to change by yourself.  
The alternative way would be specify the SG rules for ALB Ingress.

Example:    
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: "weave-scope-ingress"
  namespace: "weave"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/security-group-inbound-cidrs: 61.220.65.15/32
...(skip)
```

<br />

### **Q: Cluster Autoscler didn't trigger scale down**
A: Don't worry, normally it would cost around 10min to let the CA calculate which Pods should be moved to which Node. It really takes time bus I can't really realize why, seems the calculation not complicated at all. 



