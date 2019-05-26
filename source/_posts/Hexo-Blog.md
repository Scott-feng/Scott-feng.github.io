---
title: Hexo Blog
date: 2019-05-19 18:56:53
categories: 
  - Tech
tags: 
  - hexo
---
## Hexo 博客的搭建
参考：
- [Hexo](https://hexo.io/zh-cn/)
- [git多人协作](https://www.liaoxuefeng.com/wiki/896043488029600/900375748016320)
- [theme nexT](http://theme-next.iissnan.com/)
- [theme nexT github](https://github.com/theme-next/hexo-theme-next)

### 安装前提
- Node.js
- git

`npm install -g hexo-cli`
<!--more-->
### 建站
```bash
$ hexo init <folder>
$ cd <folder>
$ npm install
```

### 配置
__config.yml
[详细配置](https://hexo.io/zh-cn/docs/configuration)

### nexT主题安装

```bash
$ git clone https://github.com/theme-next/hexo-theme-next themes/next
```
`theme: next`

### 部署
```bash
# 推送到 dev远程分支
$ git push origin dev

$ git checkout -b branch-name origin/branch-name

# 删除分支
$ git branch -d (branchname)

```

__config.yml
部署到主分支
```yaml
deploy:
  type: git
  # repo: https://gh_token@github.com/Scott-feng/scott-feng.github.io.git
  repo: git@github.com:Scott-feng/scott-feng.github.io.git
  branch: master
```
```bash
$ hexo clean 
$ hexo g
$ hexo d
```


