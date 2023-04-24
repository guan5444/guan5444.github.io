---
title: Hugo博客相关命令
slug: hugo-blog-related-command-zedtjf
url: /post/hugo-blog-related-command-zedtjf.html
tags: []
categories:
  - post
lastmod: '2023-04-24 23:34:40'
toc: true
keywords: ''
description: >-
  hugo博客相关命令安装go安装hugod_cdd_onedrivehugohugonewsitemyblogcdmybloggitinitcdgitsubmoduleaddhttps_githubcomhugonexthugothemenextgitthemeshugothemenextcopythemeshugothemenextexamplesiteconfigyamlmoveconfigtomlconfigtomlbackuphugoservergitaddgitcommitamgitbranch
isCJKLanguage: true
---

# Hugo博客相关命令

安装go  
安装hugo  
D:  
cd D:\OneDrive\Hugo

hugo new site myblog

cd myblog

git init  
cd  
git submodule add https://github.com/hugo-next/hugo-theme-next.git themes\hugo-theme-next  
copy themes\hugo-theme-next\exampleSite\config.yaml .  
move config.toml config.toml.backup  
hugo server

git add .  
git commit -am "blog init"  
git branch -M main  
git remote add origin https://github.com/guan5444/guanblog.git  
git remote add gitee git@gitee.com:guan5444/guanblog.git  
git push -u origin "main"  
git push -u gitee "main"

cd myblog  
git submodule update --remote

hugo -F --cleanDestinationDir  
ls public

无法拉取github仓库，因为仓库含有内容，且未先clone到本地  
https://blog.csdn.net/ZCaesarK/article/details/125316158

cd public  
git init  
git remote add origin git@github.com:guan5444/guan5444.github.io.git  
git branch -M main  
git push -u origin main

‍

无法推送的话，可以强制推送

git push -f

‍
