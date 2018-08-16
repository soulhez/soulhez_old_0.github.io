---
layout: post
title:  "里程碑！Welcome to Jekyll!"
date:   2017-09-24 21:30:51 +0800
categories: Jekyll
tags: jekyll
---

个人博客搭建完成！！记录下过程中的问题作为备忘，同时也分享给大家，
让大家在遇到问题的时候多一张解决方案。

## 安装Ruby，RubyGems
安装RVM过程中在Homebrew上卡了很久，一直连接不上无法更新
Homebrew 要更新用的是curl
不使用国内的源，这是socks5科学上网出去

在zshrc中添加socks5支持
`ALL_PROXY=socks5://127.0.0.1:1080`

`brew update` 一直提示无法解析https:github.com
`brew update -verbose` 可以神奇解决!

Homwbrew成功升级，开始安装Ruby
``` shell 
$ rvm get head
$ rvm install ruby-2.3
```

## 安装Pygments
需要pip支持 [install pip](https://pip.pypa.io/en/stable/installing/)

``` shell 
$ python get-pip.py           
```

一切准备就绪，激动的输入了
``` shell 
$ jekyll new myblog
````

结果却是
``` shell 
Running bundle install in myblog...
````
没有意义的等待!!!
## 解决bundle速度慢
使用[Ruby-china]的RubyGem源替换官方源=。=
"https://gems.ruby-china.org/"
 
## 里程碑，开启Jekyll之旅
替换完成！
New jekyll site installed in myblog.


Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. 

File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. 

If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
[ruby-china]:  https://ruby-china.org/
