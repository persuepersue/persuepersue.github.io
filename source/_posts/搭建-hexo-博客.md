---
title: 搭建 hexo 博客
date: 2022-09-27
updated: 2022-09-27
---

环境：ubuntu 20.04

参考：https://hexo.io/zh-cn/docs/ https://theme-next.js.org/docs/getting-started/#Enabling-NexT

# 升级 nodejs

不然可能会遇到奇奇怪怪的问题

```sh
sudo npm install n -g
sudo n stable
hash -r
node --version
```

<!-- more -->

# 安装 hexo

```sh
sudo npm install -g hexo-cli
```



# 建站

```sh
mkdir blog
hexo init blog
cd blog
npm install	
```



## 安装 Next 主题

```sh
cd blog
npm install hexo-theme-next
npm install hexo-theme-next@latest // 更新
```

有两个配置文件：根目录下的 `_config.yml` 和 `node_modules/hexo-theme-next/_config.yml`，前者为 hexo 的配置文件，后者为 Next 的配置文件。

### 配置网站作者

Hexo config file

```yaml
# Site
author: 林芝驰
```

### 配置网站描述

Hexo config file

```yaml
# Site
description: '林芝驰的个人小站'
```

### 使能 Next 主题

Hexo config file

```yaml
theme: next
```



现在查看下网站：

```sh
hexo clean
hexo server
```

访问 http://localhost:4000/

上面配置的信息：

![image-20220925163856353](https://raw.githubusercontent.com/persuez/pictures/master/2022/09/upgit_20220925_1664095137.png)



### 配置 Next

如果要修改 Next 的配置文件，不要直接修改 `node_modules/hexo-theme-next/_config.yml`，Next 升级可能会覆盖掉配置或者有冲突导致升级不成功。参考：https://theme-next.js.org/docs/getting-started/configuration.html

```sh
cd blog
cp node_modules/hexo-theme-next/_config.yml _config.next.yml
```



## 部署

### 创建 persuepersue.github.io 仓库



### 生成 ssh key

```sh
ssh-keygen -t rsa -C "persuepersue/persuepersue.github.io"
```

保存为 /home/ql/.ssh/id_rsa_persuepersue.github.io

将 ~/.ssh/id_rsa_persuepersue.github.io.pub 贴到 github



如果一台电脑往多个 github 上提交，就需要

~/.ssh/config

```sh
host persuepersue_github.com
    Hostname github.com 
    User git 
    IdentityFile ~/.ssh/id_rsa_persuepersue.github.io
```



### 配置 github

参考

https://learnku.com/articles/48034 

https://hexo.io/zh-tw/docs/github-pages.html （zh-tw 的版本比简体中文的新）



```yaml
cd blog
git init
vim .git/config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = git@persuepersue_github.com:persuepersue/persuepersue.github.io.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main

git add .
git commit -m "hexo init"
git branch -M main
git push -u origin main
```

这里的 url 如果电脑只访问一个 github 账号，可以直接 git@github.com:persuepersue/persuepersue.github.io.git。但是如果有多个 github 账号，那就需要像上面一样。



### 部署

```sh
cd blog
mkdir .github/workflows/
vim .github/workflows/pages.yml
name: Pages

on:
  push:
    branches:
      - main  # default branch

jobs:
  pages:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Use Node.js 16.x
        uses: actions/setup-node@v2
        with:
          node-version: '16'
      - name: Cache NPM dependencies
        uses: actions/cache@v2
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

在仓库中前往 **Settings** > **Pages** > **Source**，将 branch 改为 `gh-pages`。

```sh
git add .
git commit -m "github actions 设置"
git branch -M main
git push -u origin main
```



### 访问

https://persuepersue.github.io/



# 写文章

参考：https://hexo.io/zh-tw/docs/writing

