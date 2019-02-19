---
published: true
layout: post
title: Terraform Module Private Registries - Bitbucket
author: David
modified: 2019-02-19
categories: [系統建置, Terraform]

  
comments: true
---

## Terraform Module Private Registries - Bitbucket
---

![Alt text](https://image.slidesharecdn.com/reusablecomposablebattle-testedterraformmodules-170919144930/95/reusable-composable-battletested-terraform-modules-1-638.jpg?cb=1507901802)
<br />


在網路上搜尋很久都很少看到 Terraform module 正確使用 remote private registry 的方式，所以來分享一下。

Due to less informations on the internet when I am searching `how terraform module access remote private registry`, so I decide to share this.
<br />
<br />

先講講 Terraform module 是如何運作。其實就是直接 clone 一份下來，所以等於你去 git clone。

First, we need to know how Terraform module work, that is `git clone`, Terraform will clone the module repo which you use.
<br />
<br />

而我們都知道，git分為 HTTP(S) 以及 SSH 兩種clone方式，所以在這邊兩種方式都會介紹 (居然連stackoverflow都找不到到資料 T_T)

And we know that you can clone a git repo by `HTTP(S)` and `SSH`, so I will also introduce both of these two ways.
<br />
<br />

For HTTP(S) Protocol:
```
module "base" {
    source = "git::https://bitbucket.org/davidh83110/terraform_modules.git/ec2"

}

```
<br />
<br />

For SSH Protocol:
```
module "base" {
    git::ssh://git@bitbucket.org/davidh83110/terraform_modules.git//ec2
}

```

That's all.



<br />
<br />
<div style="text-align: right;">
2019-02-19 16:11 , David in Taipei</div>

<br />
<br />
<br />