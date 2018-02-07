---
title: hexo使用指南
date: 2018-02-07 15:57:29
tags: hexo
---

使用hexo一年多时间了，今天将使用hexo使用过程做个记录 。备注（搭建平台是MAC）

<!--more-->

# 常用命令

	hexo n #写文章
	hexo g #生成
	hexo d #部署 # 可与hexo g合并为 hexo d -g
# 环境搭建 

* 安装Node.js
* 安装git（Xcode自带）
* 安装hexo

hexo 是基于Node.js的静态博客程序，使用npm 安装：

	npm install -g hexo

# GitHub

* 首先注册一个『GitHub』帐号
* 建立与你用户名对应的仓库，仓库名必须为『your_user_name.github.com』
* 添加SSH公钥到『Account settings -> SSH Keys -> Add SSH Key』

说明 git使用两种协议传输：https、git，如我的blog 的两种协议git地址如下：

git@github.com:jianli2017/jianli2017.github.io.git
https://github.com/jianli2017/jianli2017.github.io.git

SSH 作用于git协议，使用SSH 后，git协议的push、pull操作不需要输入密码。

# 初始化 

使用hexo init 命令在当前目录下初始化一个hexo项目

# 生成静态页面

cd 到你的init目录，执行如下命令，生成静态页面到 ./public/目录。

	hexo generate  

# 本地启动

执行如下命令，启动本地服务，进行文章预览调试。

	hexo server
	
# 写文章

执行new命令，生成指定名称的文章至hexo/source/_posts/postName.md。

	hexo new [layout] "postName" #新建文章
	
其中layout是可选参数，默认值为post。有哪些layout呢，请到scaffolds目录下查看，这些文件名称就是layout名称。当然你可以添加自己的layout，方法就是添加一个文件即可，同时你也可以编辑现有的layout，比如post的layout默认是hexo\scaffolds\post.md

	title: { { title } }
	date: { { date } }
	tags:
	
请注意，大括号与大括号之间我多加了个空格，否则会被转义，不能正常显示。

我想添加categories，以免每次手工输入，只需要修改这个文件添加一行，如下：

	title: { { title } }
	date: { { date } }
	categories: 
	tags: 


postName是md文件的名字，同时也出现在你文章的URL中，postName如果包含空格，必须用”将其包围，postName可以为中文。

注意，所有文件：后面都必须有个空格，不然会报错。

看一下刚才生成的文件hexo/source/_posts/postName.md，内容如下：

	title: postName #文章页面上的显示名称，可以任意修改，不会出现在URL中
	date: 2013-12-02 15:30:16 #文章生成时间，一般不改，当然也可以任意修改
	categories: #文章分类目录，可以为空，注意:后面有个空格
	tags: #文章标签，可空，多标签请用格式[tag1,tag2,tag3]，注意:后面有个空格

# fancybox
可能有人对这个Reading页面中图片的fancybox效果感兴趣，这个是怎么做的呢。
很简单，只需要在你的文章*.md文件的头上添加photos项即可，然后一行行添加你要展示的照片：

	layout: photo
	title: 我的阅历
	date: 2085-01-16 07:33:44
	tags: [hexo]
	photos:
	- http://bruce.u.qiniudn.com/2013/11/27/reading/photos-0.jpg
	- http://bruce.u.qiniudn.com/2013/11/27/reading/photos-1.jpg
经过测试，文件头上的layout: photo可以省略。

不想每次都手动添加怎么办？同样的，打开您的hexo\scaffolds\photo.md

	layout: { { layout } }
	title: { { title } }
	date: { { date } }
	tags: 
	photos: 

然后每次可以执行带layout的new命令生成照片文章：

	hexo new photo "photoPostName" #新建照片文章
## description
markdown文件头中也可以添加description，以覆盖全局配置文件中的description内容，请参考下文_config.yml的介绍。

	title: hexo你的博客
	date: 2013-11-22 17:11:54
	categories: default
	tags: [hexo]
	description: 你对本页的描述

hexo默认会处理全部markdown和html文件，如果不想让hexo处理你的文件，可以在文件头中加入layout: false。

# 文章摘要
在需要显示摘要的地方添加如下代码即可：

	以上是摘要
	<!--more-->
	以下是余下全文
more以上内容即是文章摘要，在主页显示，more以下内容点击『> Read More』链接打开全文才显示。

hexo中所有文件的编码格式均是UTF-8。

# 主题安装


安装主题的方法就是一句git命令：

	git clone https://github.com/heroicyang/hexo-theme-modernist.git themes/modernist
目录是否是modernist无所谓，只要与_config.yml文件一致即可。

安装完成后，打开hexo/_config.yml，修改主题为modernist

	theme: modernist
打开hexo/themes/modernist目录，编辑主题配置文件_config.yml：

	menu: #配置页头显示哪些菜单
	#  Home: /
	  Archives: /archives
	  Reading: /reading
	  About: /about
	#  Guestbook: /about
	excerpt_link: Read More #摘要链接文字
	archive_yearly: false #按年存档
	widgets: #配置页脚显示哪些小挂件
	  - category
	#  - tag
	  - tagcloud
	  - recent_posts
	#  - blogroll
	blogrolls: #友情链接
	  - bruce sha's duapp wordpress: http://ibruce.duapp.com
	  - bruce sha's javaeye: http://buru.iteye.com
	  - bruce sha's oschina blog: http://my.oschina.net/buru
	  - bruce sha's baidu space: http://hi.baidu.com/iburu
	fancybox: true #是否开启fancybox效果
	duoshuo_shortname: buru #多说账号
	google_analytics:
	rss:

#更新

更新hexo：

	npm update -g hexo


# 参考
* [hexo你的博客](http://ibruce.info/2013/11/22/hexo-your-blog/?utm_source=tuicool)
* [Hexo静态博客搭建+个人定制](http://blog.csdn.net/lemonxq/article/details/72676005)
* [hexo搭建的Github博客绑定域名](https://www.jianshu.com/p/cea41e5c9b2a?open_source=weibo_search)
* [初次安装git配置用户名和邮箱](https://www.cnblogs.com/superGG1990/p/6844952.html)