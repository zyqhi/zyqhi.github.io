---
title: 基于Github、Typora、jekyll建立博客写作平台
tags: jekyll Typora
typora-root-url: "/Users/mutsu/project/blog/zyqhi.github.io"
---



如今markdown基本成了轻量写作的标准方案，而且互联网比较流行的平台，诸如Github、简书、CSDN等都对markdown提供了支持。这些第三方写作平台提供写作便利的同时，也有一些不足，比如格式上不允许做定制，语法支持不完善等。

除此之外，markdown写作还有以下痛点：

1. 图片托管。采用免费的图床的话不够稳定，经常被限制，导致博客中的配图无法显示，如果付费购买对象存储空间的话，又需要付费，同时上传也比较麻烦。
2. markdown只支持有限的标记，如果想扩展标记，许多第三方写作平台是不支持的。比如以下常用的扩展标记，单纯依靠markdown就无法完成：



Success Text.
{:.success}

Info Text.
{:.info}

Warning Text.
{:.warning}

Error Text.
{:.error}



本博客采用Github+Typora+jekyll的方案从一定程度上能够解决上述问题。具体方案如下：

- Github: 博客托管平台，同时作为图床
- Typora: markdown写作软件，解决编辑器内图片预览的痛点
- jekyll: 静态博客生成，支持markdown以及扩展标记

该方案有以下优点：

- 不用第三方图床，解决插图上传、托管、编辑器内预览的苦恼，基本做到了所见即所得
- 扩展标记，满足定制与扩展需求
- Typora经过设置以后，通过Command V可以直接将剪贴板的图片存储到对应文章的路径下，并在文中插入引用链接

下文将详述该博客的结构与工作流。



## 博客结构

为了使博客结构清晰明了，易于维护，博客结构设计如下：

- 博客根目录下建立一个文件夹**media**，专门管理配图；

- 每篇文章对应一个文件夹，保存该文章内用到的配图，文件夹名称与文章名称一致，结构如下：

  ```bash
  ├── _posts
  │   ├── 2018-06-01-header-image.md
  │   ├── 2018-07-01-welcome.md
  │   ├── 2019-08-20-usbmuxd-protocol.md
  │   ├── ...
  ├── media
  │   ├── 2018-06-01-header-image
  │   ├── 2018-07-01-welcome
  │   ├── 2019-08-20-usbmuxd-protocol
  │   ├── ...
  ├── ...
  ```



## 图片引用与Typora设置

- 文章的开头yml配置中指定Typora的根路径，从而Typora能够正常显示文章采用相对位置的配图。比如本博客的设置如下：

  ``` yaml
  title: 基于Github、Typora、jekyll建立博客写作平台
  tags: jekyll Typora
  typora-root-url: "/Users/mutsu/project/blog/zyqhi.github.io"
  ```

- Typora设置里面可以设置图片使用相对位置，这样就能够在Typora中预览到图片，本博客的设置如下：

![image-20191009165642234](/../../../../../../../media/2019-10-08-using-github-typora-and-jekyll-for-blog-writing/image-20191009163031756.png)

- 图片引用全部采用相对位置，经过上述两步设置之后，在Typora中需要插入图片的位置，直接Command V可将剪贴板的图片保存到对应的目录下，同时在文章中添加对应的引用链接，示例路径如下：

  ```
  ![image-20191009165642234](/../../../../../../../media/2019-10-08-using-github-typora-and-jekyll-for-blog-writing/image-20191009163031756.png)
  ```



## 博客部署

部署这一步不需要做额外的配置。jekyll在生成静态博客时，会拷贝**media**目录下的图片到网站根目录，因此无论是本地预览还是部署到Github，图片都能够正常显示，唯一的缺憾是在博客Repo下直接查看markdown文件时，图片无法显示，不过这种场景比较少，可以略作牺牲。



## 工作流优化

上述方案建立以后，博客工作流还可以做进一步优化，首先是`typora-root-url`的设置，由于每篇文章都需要声明该配置，而且内容相同，此时可以借助插件来实现，本博客通过插件[jekyll-compose](https://github.com/jekyll/jekyll-compose)来实现。

插件安装以后，编辑博客根路径下的`_config.yml`文件，添加以下内容：

```yaml
## => Compose
jekyll_compose:
  auto_open: true
  post_default_front_matter:
    layout:
    tags:
    typora-root-url: /Users/mutsu/project/blog/zyqhi.github.io
```

然后每次新建博客时，在博客跟目录下，键入以下命令便会自动为新建的文件添加对应的设置：

```bash
bundle exec jekyll post "using github, typora and jekyll for blog writing"
```

