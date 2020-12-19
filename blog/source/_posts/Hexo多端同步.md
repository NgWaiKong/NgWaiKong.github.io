---
title: Hexo多端同步
date: 2020-04-02 12:37:48
tags:
---

# 背景：
用了Hexo作为博客方案后,在多端使用时,要拷贝源文件 & 需要确保已配置好NPM环境。有一部分的维护成本。

# 原因：
1. Hexo在Github上的master是编译过的二进制文件,并非源代码
2. master上的二进制文件,它是不可逆的，所以在其他端上无法直接使用(写博客是不行的)
3. Hexo需要Node环境

# 问题优先级：
1. 寻找一个能把源代码进行多端同步的方案
2. 解决环境配置问题

# 多端同步方案:
以下是3个方案的关键成本对比，包括：
- 维护成本：应尽量低，并能尽量保证使用体验
- 部署成本：应尽量低，能快速接入
- 费用成本：（环境）配置应尽量低

|  方案   | 维护成本 | 部署成本 | 费用成本 |
|  :----:  | :----:  | :----: | :----: |
|  硬件同步（硬盘/U盘） | 高，需要携带物理介质 | 无 | 一个U盘价格 |
| 云服务同步（百度云/腾讯云）  | 低，下载对应客户端 | 低，云客户端部署容易 | 无，不考虑会员 |
| Github  | 低，搭建Github分支 | 低，Git环境部分系统集成 |无|

因为本地使用了Mac，集成了Git，并且Git命令学习0成本，所以最终使用了Github方案

# 方案实现：
假设，两台机器，A有相关环境配置&源文件；B是全新环境。
**核心：将github.io仓库拆分2个分支，来分别管理二进制文件和源文件**
1. 将github.io拆分2个分支，来分别管理二进制文件和源文件
	- 分支1 master，存放用于展示博客的二进制文件
	- 分支2 hexo，存放hexo源文件
2. 在A机器中，将hexo相关config文件配置好，达到hexo -g/hexo -d 能直接将二进制文件发布到master分支
3. 在A机器中，将hexo源文件上传到hexo分支
4. 在B机器中，配置好环境，通过git拉取仓库，并且切换到hexo分支。
5. 在B机器中，在hexo分支中new post 正常创建博客写作。
6. 在B机器中，通过git add/commit 提交新博客在hexo分支
7. 在B机器中，执行hexo g -d生成。因为hexo config中已经配置过，hexo g后会发布到master分支。所以这步是需要执行后才能发布博客，并且必须执行。

# 细节：
1. 当用到了主题，而主题也是git的，提交前直接删到主题文件下的.git文件。
> [也可以用submodule同步](https://juejin.im/post/5af8f087f265da0b886d857a)

2. 填写gitignore 
```
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
CNAME
```

3. 新机器维护方式:
```
git pull origin hexo 
//先pull完成本地与远端的融合
npm install    //注意，这里一定要切换到刚刚clone的文件夹内执行，安装必要的所需组件，不用再init
hexo new post " new blog name"
git add source
git commit -m "XX"
git push origin hexo
hexo d -g
```

# 环境配置问题思路:
统一在私有云服务器上实现生成和发布，客户端不需要关心环境问题。
1. 私有云服务器上搭建Node环境
2. 通过协议，讲写好的博客上传到私有云
3. 私有云直接实现hexo发布


# 引用：
> https://zhuanlan.zhihu.com/p/26625249
> https://zomfice.github.io/2018/02/25/Hexo%E5%8D%9A%E5%AE%A2%E6%90%AD%E5%BB%BA-%E4%B8%89-%E5%A4%9A%E8%AE%BE%E5%A4%87%E5%90%8C%E6%AD%A5/
> https://www.zhihu.com/question/21193762/answer/79109280
> https://shencai.xyz/2019/05/08/hexo-github%E5%A4%9A%E7%AB%AF%E5%90%8C%E6%AD%A5/