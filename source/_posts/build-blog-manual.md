---
title: hexo搭建github博客
categories: others
---

# Hexo搭建博客

## 安装Node.js

[Node.js下载地址](https://nodejs.org/en/download/)

[Node.js安装教程](https://www.npmjs.cn/getting-started/installing-node/)

* 安装完成之后检测是否安装成功

``` bash
node -v
```

**npm是Node.js的包管理工具**

* 安装Node.js的时候，已经顺带安装好了npm，检测npm是否安装成功

``` bash
npm -v
```

## 安装Hexo

``` bash
-- 安装
npm install -g hexo-cli
-- 初始化博客
hexo init blog
-- 生成hexo
hexo g
-- 本地启动
hexo s
-- 部署
hexo d
```

高阶命令

``` bash
-- 升级hexo
npm update hexo -g
-- 指定端口
hexo server -p 5000
-- 自定义ip
hexo server -i 192.168.1.1

-- 新建文档 layout默认为post
hexo new [layout] <title>
```

启动完成之后就可以通过 http://localhost:4000 进行访问；

> [Hexo官网地址](https://hexo.io/)
> [Hexo主题](https://hexo.io/themes/)
> [Hexo指令](https://hexo.io/zh-cn/docs/commands.html#new)
> [Hexo docs](https://hexo.io/docs/)
> [Markdown语法](https://www.markdown.xyz/basic-syntax/)


## GitHub结合

* 安装git部署插件

``` bash
npm install hexo-deployer-git --save
```

* 修改_config.yml配置文件

``` yaml
deploy:
  type: git
  repo: https://github.com/vandyzhou/vandyzhou.github.io.git
  branch: master
```

* 然后部署hexo到github上

``` bash
hexo g
hexo d
```

然后访问 https://vandyzhou.github.io 即可浏览

