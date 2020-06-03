---
author: David
categories:
- AWS
- Kubernetes
# description: "defwefgegewgew"
comments: true
date: "2020-05-14"
modified: "2020-05-14"
published: true
title: EKS External-DNS Cross Account Route53 Setup
images: ["https://live.staticflickr.com/65535/49908288961_c380509717_b.jpg width=10"]
url: /aws/kubernetes/系統建置/terraform/2020/05/14/eks-external-dns.html
---


External-DNS 是一款可以讓我們建立 ALB Ingress 的時候，自動將 ALB DNS 在 Route53 建立一組 DNS 的實用套件。

當然他也支援在 GCP/Azure 上面建立，只是這邊就只介紹 AWS 上如何做設定，以及如果 Route53 在另一個 Account 的話要怎麼做。 在 EKS 1.14 以上使用了 OIDC ，所以如果跨帳號就會需要使用 OIDC 做授權。

External-DNS 官方介紹：  

> Inspired by Kubernetes DNS, Kubernetes' cluster-internal DNS server, ExternalDNS makes Kubernetes resources discoverable via public DNS servers. Like KubeDNS, it retrieves a list of resources (Services, Ingresses, etc.) from the Kubernetes API to determine a desired list of DNS records. Unlike KubeDNS, however, it's not a DNS server itself, but merely configures other DNS providers accordingly—e.g. AWS Route 53 or Google Cloud DNS.
>
> In a broader sense, ExternalDNS allows you to control DNS records dynamically via Kubernetes resources in a DNS provider-agnostic way.
> 

- [官方 Github Repository](https://github.com/kubernetes-sigs/external-dns)

<br />

## Prepare
***

- ALB Ingress Controller needs to be installed on EKS cluster.
<br />

  如果沒有安裝的話可以參考我的文章進行安裝, [文章連結](https://blog.davidh83110.com/aws/kubernetes/%E7%B3%BB%E7%B5%B1%E5%BB%BA%E7%BD%AE/terraform/2020/03/30/eks-install.html)

<br />
<br />

## Setup with Single AWS Account
---

其實只有一個帳號的話，設定上很容易，按照官方的說明就可以了。
<br />


- 建立授權給 Route53 的 IAM Policy, 並指配給 EKS Service Role  

  ```
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "route53:ChangeResourceRecordSets"
        ],
        "Resource": [
          "arn:aws:route53:::hostedzone/*"
        ]
      },
      {
        "Effect": "Allow",
        "Action": [
          "route53:ListHostedZones",
          "route53:ListResourceRecordSets"
        ],
        "Resource": [
          "*"
        ]
      }
    ]
  }
  ```

<br />
- 安裝 External-DNS 在 EKS / 驗證安裝  

  [參考官方安裝教學](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md)

  或參考附錄我的 YAML, 修改帳號及 Role Name, 再 Apply to cluster 就可以。 [附錄](#附錄)

  也可以 `--domain-filter` 指定要控制的 Domain, 避免影響到其他 Domain.

<br />
<br />

## Setup with Cross-Account (Route53 Zone and EKS in different account)
---
這邊就會比較複雜, 也是這篇要講的重點。 在 EKS 1.14 以上使用 OIDC 的部分，設定上會比較不一樣。

<br />

**環境假設:**  
有兩個 AWS Accounts, 分別是 Root Account & EKS Account.  
`Root Account`: 管理 Route53 Hosted Zone  
`EKS Account`: 管理 EKS Cluster

<br />

**建立 Route53 Permission:**  
我們需要建立這個授權建立 Record 的 IAM Policy 在 `Root Account`  

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

<br />

**建立 OIDC Identity Provider:**  
要在 `Root Account` 做, 因為我們是要把 EKS Account 連結到 Root Account。

請在 `Root Account` 建立 IAM Identity Provider  
> **Provuder Type** - 請選擇 OpenID Connect  
> **Provider URL** -  請填入 EKS Account EKS Cluster 的 OIDC URL。  
> **Audience** - 請填入 `sts.awsamazon.com`

![text](https://live.staticflickr.com/65535/49907767258_345e5bf7cb_k.jpg)

<br />

**建立 IAM Role:**  
接下來需要在 `Root Account` 建立一個 IAM Role 給剛剛建立的 Identity Provider 套用。  

請選擇 `Web Identity` 類型建立 IAM Role, 並選擇剛剛的 Identity Provider & Audience。

![text](https://live.staticflickr.com/65535/49908580392_88e700df71_k.jpg)

<br />

**修改 Trusted Relationship (optional):**  
我們要對 trusted relationship 做一些修改。 (直接編輯 trusted relationship YAML)
<br />

- 將 `aud` 改成 `sub`  
- 將 `namespace` 改成 `serviceaccount`

<br />
改完之後應該會像這樣  

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::0729292222:oidc-provider/oidc.eks.ap-northeast-1.amazonaws.com/id/81309C8WWMNK2BF580A65EB4099B"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-northeast-1.amazonaws.com/id/81309C8WWMNK2BF580A65EB4099B:sub": "system:serviceaccount:default:external-dns"
        }
      }
    }
  ]
}
```

<br />

**安裝 External-DNS:**  
[參考官方安裝教學](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md)

或參考[附錄](#附錄)，直接修改我的參考 YAML 裡的帳號跟 Role Name, 然後 Apply to Cluster.  

> 可以 `--domain-filter` 指定要控制的 Domain, 避免影響到其他 Domain.

<br />

**跟 Single Account的不同:**   
在附錄裡的 YAML , Account ID & Role Name 請填入 `Root Account` 的 ID 跟 Role Name. (我們前幾步驟建立的那個 Role)

<br />
<br />

## 附錄
---  
- External-DNS YAML  

  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: external-dns
    # If you're using Amazon EKS with IAM Roles for Service Accounts, specify the following annotation.
    # Otherwise, you may safely omit it.
    annotations:
      # Substitute your account ID and IAM service role name below.
      eks.amazonaws.com/role-arn: arn:aws:iam::{{ account_id }}:role/{{ iam_service_role_name }}
  ---
  apiVersion: rbac.authorization.k8s.io/v1beta1
  kind: ClusterRole
  metadata:
    name: external-dns
  rules:
  - apiGroups: [""]
    resources: ["services","endpoints","pods"]
    verbs: ["get","watch","list"]
  - apiGroups: ["extensions"]
    resources: ["ingresses"]
    verbs: ["get","watch","list"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list","watch"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1beta1
  kind: ClusterRoleBinding
  metadata:
    name: external-dns-viewer
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: external-dns
  subjects:
  - kind: ServiceAccount
    name: external-dns
    namespace: default
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: external-dns
  spec:
    strategy:
      type: Recreate
    selector:
      matchLabels:
        app: external-dns
    template:
      metadata:
        labels:
          app: external-dns
        # If you're using kiam or kube2iam, specify the following annotation.
        # Otherwise, you may safely omit it.
        annotations:
          iam.amazonaws.com/role: arn:aws:iam::{{ account_id }}:role/{{ iam_service_role_name }}
      spec:
        serviceAccountName: external-dns
        nodeSelector:
          app: system
        containers:
        - name: external-dns
          image: registry.opensource.zalan.do/teapot/external-dns:latest
          args:
          - --source=service
          - --source=ingress
          - --provider=aws
          - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
          - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
          - --registry=txt
          - --txt-owner-id=my-hostedzone-identifier
        securityContext:
          fsGroup: 65534 # For ExternalDNS to be able to read Kubernetes and AWS token files
  ```