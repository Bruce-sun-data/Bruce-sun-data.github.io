---
title: hexo更换主题
date: 2021-05-13 16:08:50
tags: hexo插图
typora-root-url: ..
---

# Hexo设置插图

hexo可以很好的结合Typora进行博客文章的插图处理，就只需要两步操作

## 设置图片根目录

打开`格式(O)-图像-设置图片根目录`，选择`source`文件夹（就是`_posts`上一级），确定，这时候回到文件中，可以看到在文章头部出现了这么一行字：![设置图片根目录](./images/hexo更换主题/image-20210513163932292.png)

## Typora配置

在Typora中设置图像粘贴位置

打开`格式(O)-图像-全局图像设置`，在“插入图片时…”选择**复制到指定路径**，然后在下面写入`../images/${filename}`（`$`中的参数是以文件名命名的文件夹），并勾选“优先使用相对路径”。

![typora更改路径](./images/hexo更换主题/image-20210513164424278.png)

完成上面这两个步骤就可以在自己hexo的主页中看到自己上传的图片了。

# 设置主题

首先去hexo的主题官网[Themes | Hexo](https://hexo.io/themes/)选一个主题，这里就介绍最简单的NEXT主题叭。

## 安装主题

进入hexo对应的目录，使用git运行以下命令

```
git clone https://github.com/iissnan/hexo-theme-next themes/next
```

git结果如下图

![下载主题](./images/hexo更换主题/image-20210513184904009.png)

## 设置主题

修改当前目录下的`_config.yml`文件，找到 theme 字段，并将其值更改为 next。

```
theme: next
```

然后使用git重新部署网站

```
hexo d -g
```

但是出现了如下错误 `{% extends ‘_layout.swig‘ %} {% import ‘_macro/post.swig‘ as post_template %}`

![next出错](./images/hexo更换主题/image-20210513192500960.png)



之后在网上找到该问题的答案，用npm安装swig。

```
npm install hexo-renderer-swig --save
```

之后就可以看到next的主题风格了。

![next主页](./images/hexo更换主题/image-20210513194508921.png)

## 修改网站中的配置文件

在当前目录下进入对`_config.yml`文件内容进行修改。