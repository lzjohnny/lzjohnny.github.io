# HEXO 搭建及使用

## NPM 基础环境
 - 安装 NPM
 - 任意目录执行 `npm install -g hexo-cli` 全局安装 hexo-cli

## HEXO 环境
 - 下载博客模板后进入该目录
`git clone -b hexo-framework git@github.com:lzjohnny/lzjohnny.github.io.git`
 - 下载主题到 themes\aoba 目录
`git clone git@github.com:lzjohnny/hexo-theme-aoba.git themes/aoba`
 - 安装环境 `npm install`

## 启动 HEXO
`hexo server`

## HEXO 使用
```
hexo new "My New Post"
# 写文章......
hexo server
hexo generate
hexo deploy
```
