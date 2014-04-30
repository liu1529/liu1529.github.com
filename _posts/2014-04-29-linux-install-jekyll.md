---
layout: post
title: linux安装jekyll
comments: true
tags: [linux, jekyll, github]
---


# jekyll 安装
以下内容主要参考[jekyll中文官网](http://jekyllcn.com/)。笔者使用的OS为ubuntu。

## ruby 安装
jekyll依赖Ruby，RubyGems，如果计算机没有安装，应先使用

    sudo apt-get install ruby

安装ruby，这儿必须注意：

ruby的版本必须在1.9.1以上，否则安装jekyll会出错，笔者的OS默认安装的是1.8，所以必须使用`sudo apt-get install ruby1.9.1` 安装ruby
## jekyll 安装
安装完ruby后，使用

    gem install jekyll 

安装jekyll

## jekyll 示例
 
安装完成jekyll后，使用如下命令

    jekyll new mybolg
    cd mybolg
    jekyll serve

创建一个简单的示例。在浏览器中输入：`localhost:4000`即可浏览生成的博客界面。
修改_posts文件夹下面的文件，即可修改发表的文章。参考_posts文件夹下面的文件，使用类似的文件命名方式，即可新建文章。

## github page的使用

完成上面后，已经可以jekyll新建和修改文章。下面讲述如何使用github page页面，建立自己的个人主页。

首先，在github上建立新建一个仓库，仓库的名称为`username.github.io`。然后，将仓库clone到本地，在本地进行一次push。最后，在github的仓库设置界面，找到GitHub Pages选项，在下面可以看到自己的主页地址，如果设置正确，主页地址应该为`username.github.io`的形式。在浏览器中输入`username.github.com`或`username.github.io`就可以看见自己的主页。

