+++

title = "将Hugo博客部署到Github Pages"
date = 2022-02-12T19:34:01+08:00
slug= "hugo_blog"
description = "利用Github Pages部署Hugo生成的静态博客页面"
tags = [ "Hugo" ]
categories = [ "Tech" ]
image = ""

+++

## 关于Hugo

Hugo的便利之处在于，用户只需编辑markdown文档，Hugo会自动将markdown文档转换为网页。Hugo根据存放于content文件夹中的用户markdown文件，生成网页源文件，并存放于public文件夹中。

将博客部署到GitHub Pages，只需将Hugo生成的public文件夹推送到GitHub仓库。

## 关于Github Pages

官方文档定义：
> GitHub Pages is a static site hosting service that takes HTML, CSS, and JavaScript files straight from a repository on GitHub, optionally runs the files through a build process, and publishes a website.

[Github Pages](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages) 有两种形式，个人/组织页面和项目页面，两者访问时的url不同，为了能够使用`https://<USERNAME|ORGANIZATION>.github.io/`访问个人博客，应当设置成个人页面。

- User/Organization Pages (`https://<USERNAME|ORGANIZATION>.github.io/`)
- Project Pages (`https://<USERNAME|ORGANIZATION>.github.io/<PROJECT>/`)

## 创建User Page

在Github新建仓库时，个人页面的创建和项目页面不同：

- 用于个人页面的仓库必须被用户（而不是组织）所有，并将仓库命名为`<username>.github.io`

- 个人页面的源文件放置于仓库的默认分支中，项目页面需存放在特定分支

创建仓库时需要注意，免费用户创建的页面仓库必须设置为Public

![新建仓库](https://cdn.jsdelivr.net/gh/fmpic/imghost/20220214110340.png)

选择 **Initialize this repository with a README** ，完成仓库创建


## 将源文件推送到仓库

在创建的仓库中复制远程仓库地址

![仓库地址](https://cdn.jsdelivr.net/gh/fmpic/imghost/20220214111325.png)

在Hugo生成的文件夹中，在终端中输入

```powershell
hugo  #生成网页源文件
cd public  #生成的源文件存放在public文件夹中，只需将该文件夹推送到所创建的仓库中
git init  #git初始化
git remote add origin git@github.com:<username>/<username>.github.io.git  #关联远程仓库
git add .
git commit -m "first commit"  #在本地提交更改
git push -u origin master  #将更改推送到远程仓库
```

此时GitHub仓库中拥有main和master两个分支，其中main分支是创建仓库是自动生成的默认分支，mater分支由本地推送，即博客的网页源文件

GitHub Pages的个人页面默认从main分支提取网页源文件，所以还需要在仓库的`settings-pages-source`中将分支改为master分支

![设置页面分支](https://cdn.jsdelivr.net/gh/fmpic/imghost/20220214111729.png)

之后即可通过`<username>.github.io`访问博客页面