---
author: David
categories:
- Terraform
comments: true
date: "2019-02-03T00:00:00Z"
modified: "2019-02-19"
published: true
title: Terraform Module Private Registries - Bitbucket
iamges: ["https://farm8.staticflickr.com/7824/46420454484_fa00ae8615_h.jpg"]
url: /系統建置/terraform/2019/02/03/terraform-module-bitbucket.html
---

在網路上搜尋很久都很少看到 Terraform module 正確使用 remote private registry 的方式，所以來分享一下。  
Due to less information on the internet when I am searching `how terraform module access remote private registry`, so I decide to share this.


先講講 Terraform module 是如何運作。其實就是直接 clone 一份下來，所以等於你去 git clone。  
First, we need to know how Terraform module work, that is `git clone`, Terraform will clone the module repo which you use.

而我們都知道，git分為 HTTP(S) 以及 SSH 兩種clone方式，所以在這邊兩種方式都會介紹 (居然連stackoverflow都找不到到資料 T_T)  
And we know that you can clone a git repo by `HTTP(S)` and `SSH`, so I will also introduce both of these two ways.  

For HTTP(S) Protocol:
```py
module "base" {
    source = "git::https://bitbucket.org/davidh83110/terraform_modules.git/ec2"
}
```

For SSH Protocol:
```py
module "base" {
    source = "git::ssh://git@bitbucket.org/davidh83110/terraform_modules.git//ec2"
}
```


> 在 SSH clone 方式裡，請使用 `//` 雙斜線來表示目錄 (i.e. //ec2)  
> Please use a double slash (`//`) to clone a directory (i.e. //ec2).


That's all.

<br />
<div style="text-align: right;">
2019-02-19 16:11 , David in Taipei</div>

<br />
<br />
<br />
