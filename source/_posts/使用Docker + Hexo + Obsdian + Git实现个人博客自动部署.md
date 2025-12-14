---
title: 使用Docker + Hexo + Obsdian + Git实现个人博客自动部署
date: 2025-12-14 16:04:46
tags: []
categories: []
---


这件事想做很久了。
阿里云99/年服务器已经闲置两年了，终于想起来用了 hh

# 一、拥有一台安装docker的服务器

在服务器上安装docker很简单 这里不在赘述

# 二、开始生成Hexo镜像和容器

首先确定你博客app的目录 我这里使用的是 `/app/hexo/`

未来的结构如下

``` 
/opt/hexo
├── docker-compose.yml
├── Dockerfile
└── blog/              # 博客实际源码（文章都在这里）
    ├── _config.yml
    ├── source/
    ├── themes/
    └── ...

```

然后创建Dockerfile文件

`vim /app/hexo/Dockerfile`

```
FROM node:lts

# Hexo 工作目录
WORKDIR /blog

# 安装 hexo-cli
RUN npm install -g hexo-cli

# 默认 CMD：每次启动自动安装依赖 + 启动服务
CMD npm install && hexo server -i 0.0.0.0
```

其次 创建docker-composer.yml 来确定容器创建的规则 

`vim /app/hexo/docker-compose.yml`

```
version: "3.8"

services:
  hexo:
    build: .
    container_name: hexo-blog
    ports:
      - "4000:4000"
    volumes:
      - ./blog:/blog       # 博客源代码挂载
    restart: unless-stopped

```

第四步，初始化hexo博客，在容器中初始化，文件会生成在挂载的文件目录下

```bash
cd /opt/hexo
docker compose build
docker compose run --rm hexo hexo init .
```

移动到 blog 文件夹：

```bash
mkdir blog mv _config.yml package.json scaffolds source themes blog/
```

（如果其他文件生成，也移动进去）

然后删除 root 目录残留：`rm -f package-lock.json`

第五步，启动Hexo容器

`docker compose up -d`

访问 我的服务器地址 `http://你的IP:4000`

# 三、设置自定义主题

主题 目录是放在 `/app/hexo/blog/themes` 下的

进入 /app/hexo/blog/ 目录下 从git clone 主题 到 `themes/hacker`

`git clone https://github.com/theme-hacker/hexo-theme-hacker themes/hacker`

然后修改配置 `_config.yml` 指定主题

`theme: hacker`

最后重启容器即可

`docker compose restart`

# 四、我们最初的愿景

### 发布流程是：

1. 本地或服务器写文章
    
2. `git add source/_posts/*.md`
    
3. `git commit -m "new post"`
    
4. `git push`
    
5. GitHub Webhook 触发服务器脚本
    
6. 脚本执行：
    

```bash
git pull
docker compose exec hexo hexo clean 
docker compose exec hexo hexo generate
```

``` perl
Windows（写文章）
   ↓ git push
GitHub Repo
   ↓ Webhook
服务器（Docker + Hexo）
   ↓
hexo clean / generate
   ↓
生成静态博客

```

也就是说我需要将服务其中的blog文件夹全部托管在github，然后windows和服务器公用使用这个git仓库来实现博客文章的同步。

首先编辑 `.gitignore` 文件 不要把一些插件的配置文件放进去。

```
node_modules/
public/
.deploy_git/
db.json
```

然后在服务器 `/app/hexo/blog/` 目录下创建使用git初始化这个文件夹，将其设置为一个git仓库

`git init`

同时你还需要在github上创建一个repository作为远程仓库 然后绑定本地仓库

`git remote add origin git@github.com:wcf178/hexo-blog.git`

然后 `git add .` `git commit -m "init"` 发现报错了 原来是我还没有设置用户和邮箱

```
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

完了之后 继续 commit 然后 push 欸 又报错了

```
The authenticity of host 'github.com (20.205.243.166)' can't be established.
ECDSA key fingerprint is SHA256:p2QAMXNIC1TJYWeIOttrVc98/R1BUFWu3/LiyKgUfQM.
ECDSA key fingerprint is MD5:7b:99:81:1e:4c:91:a5:0d:5a:2e:2e:80:13:3f:24:ca.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'github.com,20.205.243.166' (ECDSA) to the list of known hosts.
Permission denied (publickey).
fatal: Could not read from remote repository.
```
源赖氏 没有配置ssh key

那就生成 直接来吧！

`ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`

然后将公钥添加到GitHub

`cat ~/.ssh/id_rsa.pub`

测试链接 

`ssh -T git@github.com`

验证输出

`Hi <your-username>! You've successfully authenticated, but GitHub does not provide shell access.`

重新推送 

`git push -u origin master`

# 五、编写自动部署脚本

`vim /app/hexo/deploy.sh`

直接写以下的内容

```
#!/bin/bash
set -e

cd /app/hexo/blog

echo ">>> Pull latest code"
git pull origin main

echo ">>> Generate hexo"
docker compose exec -T hexo hexo clean
docker compose exec -T hexo hexo generate

echo ">>> Done"
```

赋权

`chmod +x /app/hexo/deploy.sh`

# 六、webhook实现push后 github调用服务器的deploy.sh

原本的方案是使用现有的webhook镜像+GitHub webhook实现，但是发现推荐的webhook都挂了
所以就换为 **GitHub Action + SSH** 来实现 
实际上还有 **Nginx + shell** 的方案 似乎也可以，只不过我还没有尝试。先说下第一种方法的实现。

首先在服务器上生成专门用于CI的key

`ssh-keygen -t ed25519 -C "github-actions-hexo" -f ~/.ssh/hexo_ci`

然后把公钥加入服务器授权

```
cat ~/.ssh/hexo_ci.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

然后 进入GitHub 仓库 → Settings → Secrets and variables → Actions

查看私钥

`cat ~/.ssh/hexo_ci`

新增 Secrets：

| Name              | Value         |
| ----------------- | ------------- |
| `SSH_PRIVATE_KEY` | 刚才复制的私钥       |
| `SSH_HOST`        | 服务器 IP 或域名    |
| `SSH_USER`        | `root`（或你的用户） |
| `SSH_PORT`        | `22`（如果没改）    |
然后创建 github工作流

`vim .github/workflows/deploy.yml` 

注意 这的文件目录不可以更改，github是只认这里的yml文件

编辑内容

```yml
name: Deploy Hexo Blog

on:
  push:
    branches:
      - master #这里需要替换为你实际的分支

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -p ${{ secrets.SSH_PORT }} ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts

      - name: Deploy to server
        run: |
          ssh -p ${{ secrets.SSH_PORT }} ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }} << 'EOF'
            cd /app/hexo
            ./deploy.sh
          EOF

```

然后提交并推送文件 就可以在github仓库的action里看到有一个action在运行。

至此 完整的工作流已经构建完毕 

最后可以测试一下整个流程 ：

在windows下 的本地仓库 （如果你还没有clone 先clone到本地） 新建一篇文章或者更改现在的文章 提交并推送，最后访问博客页面 发现文章已经新增或者更改。

# 七、结合obsidian使用git插件实现完美的写作-推送流程

windows编辑文章总要一个合适的编辑器。之前总有听闻obsidian的大名，两年前下载了，但是基本不用，现在终于可以有个理由用上了。

单单是 obsidian 似乎还是需要先写文章，然后使用git管理工具推送，有没有更加方便的一站式流程呢？这让我想到了vs code中集成了git的提交推送等大部分功能。（当然如果熟悉了md文档格式的编写  可以直接使用vs code作为本地编辑器。）

然后我就搜索了一下git插件，果然是有的。将 `source/post` 文件作为obsidian仓库，然后将 `.obsidian/` 加入到.gitignore 文件，最后简单配置下obsidian 的git插件，就大功告成了。

本地使用obsidian编辑 -> 写完之后顺手推送到github备份，github action 发送服务器信息 自动部署。