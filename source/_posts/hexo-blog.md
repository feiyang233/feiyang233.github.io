---
title: GitHub-Hexo-NexT 免费搭建自己的博客
date: 2019-03-15 11:05:30
tags: hexo
categories: 学习
---
## 关于写博客
2018 年我在 GitHub 上看到[强哥的博客](http://blog.dploop.org/)，当时为之心动然后 Fork 了那个 repository。后来搬砖沦为社会人就断更了，在工作之后看到[奥哥的博客](https://wsgzao.github.io/)，在他的安利之下，把之前的 Jekyll 的博客换成如今的 Hexo-NexT，打算记录下工作的点滴，学习的心得。一是分享，二是方便自己回顾和查阅。<!--more-->  

## GitHub
如果没有 GitHub 账号，需要去官网 https://github.com/ 注册一个账号。账号注册好之后，需要创建一个 repository，名称格式为 xxx.github.io  例如我的 GitHub 名称为 fainyang, 浏览器显示的 URL 为 https://github.com/fainyang 那么我新建的 repository 就是 fainyang.github.io    

![github](/img/2019/github.png)  

申请完账号，创建了 repository 之后，下一步就是在自己的电脑上安装 Hexo。
## Hexo
Hexo——快速、简洁且高效的博客框架。官网 https://hexo.io/zh-cn/ 文档还有视频讲解如何安装  
### 安装前提
在安装 Hexo 之前，需要安装 Git 与 Node.js 请参考 [这里](https://hexo.io/zh-cn/docs/#%E5%AE%89%E8%A3%85)   
### 开始安装
```shell
npm install hexo-cli -g  
npm install hexo --save
hexo init <folder>   # <folder> 自己取个文件夹名字
cd <folder>          # 移动到 <folder> 文件夹，不清楚当前路径，可以执行 Mac:pwd  Windows: chdir
npm install          #feiy-mac:Coding feiy$ pwd    /Users/feiy/Coding
```
新建完成后，指定文件夹的目录如下：
```
.
├── _config.yml    #站点配置文件,主要就是修改这个文件
├── package.json
├── scaffolds      #模板文件夹
├── source         #存放文章，图片的文件夹
|   ├── _drafts
|   └── _posts
└── themes         #主题文件夹，等会下载 NexT 主题
```

###  这是我的 _config.yml 配置 
```yml
# Hexo Configuration
## Docs: https://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
title: feiyang's blog  
subtitle: 费洋的博客
description: 佛系咸鱼
keywords:
author: feiyang
language: zh-CN     #网址的显示语言
timezone:

# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: https://fainyang.github.io      #需要修改为自己的网址
enforce_ssl: fainyang.github.io
root: /
permalink: post/:title/
permalink_defaults:

# Directory
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:

# Writing
new_post_name: :title.md 
 # File name of new posts
default_layout: post
titlecase: false # Transform title into titlecase
external_link: true # Open external links in new tab
filename_case: 0
render_drafts: false
post_asset_folder: false
relative_link: false
future: true
highlight:
  enable: true
  line_number: true
  auto_detect: false
  tab_replace: 
  
# Home page setting
# path: Root path for your blogs index page. (default = '')
# per_page: Posts displayed per page. (0 = disable pagination)
# order_by: Posts order. (Order by date descending by default)
index_generator:
  path: ''
  per_page: 10
  order_by: -date
  
# Category & Tag
default_category: uncategorized
category_map: 
tag_map: 

# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss

# Pagination
## Set per_page to 0 to disable pagination
per_page: 10
pagination_dir: page

# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: next

# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: https://github.com/fainyang/fainyang.github.io.git  #部署上传的地址，需要修改为自己的
  branch: master
  message: new hexo_blog
  

# Others
archive_generator:
  per_page: 10
  yearly: true
  monthly: true

tag_generator:
  per_page: 10
```
### Hexo 常用命令
官网详细的介绍 https://hexo.io/zh-cn/docs/commands

```shell
命令缩写
hexo n "文章标题" == hexo new "文章标题" #新建文章
hexo p == hexo publish
hexo g == hexo generate #生成
hexo s == hexo server   #启动服务预览
hexo d == hexo deploy   #部署
```
### 本地运行网站
```shell
hexo s  # 当前路径要在之前创建的 hexo init <folder>  这个 <folder> 下
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```
在浏览器输入 `http://localhost:4000` 就可以看到初始的网站 
## NexT
Elegant Theme for Hexo——精于心，简于形。 官网 http://theme-next.iissnan.com/

### 安装 Next
```shell
# 当前路径要在之前创建的 hexo init <folder>  这个 <folder> 下 
# 查看当前路径 Mac:pwd  Windows: chdir
cd <folder>
git clone https://github.com/theme-next/hexo-theme-next themes/next
```
git clone 完成后，指定文件夹的目录如下：
```
.
├── _config.yml    #站点配置文件，主要就是修改这个文件
├── package.json
├── scaffolds      #模板文件夹
├── source         #存放文章，图片的文件夹
|   ├── _drafts
|   └── _posts
└── themes         #主题文件夹，等会下载 NexT 主题
    ├── landscape
    └── next
        └── _config.yml  #主题配置文件，修改博客主题样式等
```
### 主题配置文件 _config.yml
```yml
#  菜单显示项
menu: 
  home: / || home
  about: /about/ || user
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive

# Schemes 主题选择，我用的是 Gemini
#scheme: Muse
#scheme: Mist
#scheme: Pisces
scheme: Gemini
```
此时可以在本地预览一下运行命令 `hexo s`
### 部署到 GitHub 
部署前，需要修改配置文件里面的 git 地址
```yml
deploy:
  type: git
  repo: https://github.com/yourname/yourname.github.io.git  #部署上传的地址，需要修改为自己的
  branch: master
  message: new hexo_blog
```
修改好 _config.yml 里的 deploy 模块后，就可以开始进行部署，上传到 GitHub
```shell
npm install hexo-deployer-git --save
hexo clean
hexo generate
hexo deploy
```
第一次部署需要输入自己的 GitHub 账号和密码。后面更新部署时就不需要再次输入密码了。
```shell
feiy-mac:fainyang.github.io feiy$ hexo clean     #清除缓存
INFO  Deleted database.
INFO  Deleted public folder.
feiy-mac:fainyang.github.io feiy$ hexo generate  #生成文件
INFO  Start processing
INFO  Files loaded in 902 ms
INFO  Generated: sitemap.xml
INFO  Generated: atom.xml
INFO  Generated: index.html
INFO  Generated: tags/index.html
INFO  Generated: categories/index.html
INFO  Generated: archives/index.html
INFO  Generated: about/index.html
INFO  Generated: img/combination1.png
INFO  Generated: images/algolia_logo.svg
INFO  Generated: img/combination2.png
INFO  Generated: img/404-southpark.jpg
INFO  Generated: img/favicon.ico
INFO  Generated: img/gay.jpeg
INFO  Generated: images/apple-touch-icon-next.png
INFO  Generated: img/old_driver.jpg
INFO  Generated: img/qq-qrcode.png
INFO  Generated: img/zjxc.jpg
INFO  Generated: images/cc-by-nc-nd.svg
INFO  Generated: images/avatar.gif
INFO  Generated: images/cc-by-nc-sa.svg
INFO  Generated: images/cc-by-nc.svg
INFO  Generated: images/cc-by-sa.svg
INFO  Generated: images/cc-by-nd.svg
INFO  Generated: images/cc-by.svg
INFO  Generated: images/favicon.ico
INFO  Generated: images/quote-l.svg
INFO  Generated: images/cc-zero.svg
INFO  Generated: images/loading.gif
INFO  Generated: images/logo.svg
INFO  Generated: images/searchicon.png
INFO  Generated: images/favicon-16x16-next.png
INFO  Generated: images/placeholder.gif
INFO  Generated: images/favicon-32x32-next.png
INFO  Generated: images/quote-r.svg
INFO  Generated: images/feiy.gif
INFO  Generated: archives/2019/03/index.html
INFO  Generated: archives/2018/02/index.html
INFO  Generated: post/hexo-blog/index.html
INFO  Generated: img/work/6.jpg
INFO  Generated: archives/2018/08/index.html
INFO  Generated: categories/学习/index.html
INFO  Generated: archives/2018/11/index.html
INFO  Generated: archives/2018/07/index.html
INFO  Generated: img/robot/run.png
INFO  Generated: tags/python/index.html
INFO  Generated: categories/生活/index.html
INFO  Generated: tags/hexo/index.html
INFO  Generated: tags/工作/index.html
INFO  Generated: archives/2018/04/index.html
INFO  Generated: archives/2018/03/index.html
INFO  Generated: tags/links/index.html
INFO  Generated: post/Canteen/index.html
INFO  Generated: tags/NUS/index.html
INFO  Generated: img/2019/github.png
INFO  Generated: post/Onboarding/index.html
INFO  Generated: post/Combination/index.html
INFO  Generated: post/Robot/index.html
INFO  Generated: post/blog/index.html
INFO  Generated: post/Master/index.html
INFO  Generated: post/links/index.html
INFO  Generated: post/timestampdiff/index.html
INFO  Generated: img/time3.png
INFO  Generated: img/start.jpg
INFO  Generated: img/robot/tencent.png
INFO  Generated: img/work/8.JPG
INFO  Generated: img/work/7.JPG
INFO  Generated: img/avatar-icon.jpg
INFO  Generated: img/time1.png
INFO  Generated: img/wx-qrcode.png
INFO  Generated: archives/2018/index.html
INFO  Generated: archives/2019/index.html
INFO  Generated: img/146228364162795.jpg
INFO  Generated: lib/font-awesome/HELP-US-OUT.txt
INFO  Generated: css/main.css
INFO  Generated: img/canteen/bird.jpg
INFO  Generated: img/combination3.png
INFO  Generated: img/time4.png
INFO  Generated: img/path.jpg
INFO  Generated: img/time2.png
INFO  Generated: img/work/10.JPG
INFO  Generated: img/work/5.JPG
INFO  Generated: img/work/13.JPG
INFO  Generated: img/work/2.JPG
INFO  Generated: img/work/1.JPG
INFO  Generated: lib/font-awesome/bower.json
INFO  Generated: img/robot/ditou.jpg
INFO  Generated: img/canteen/contact.png
INFO  Generated: img/robot/happy.jpg
INFO  Generated: img/robot/poor.jpeg
INFO  Generated: img/robot/miaohui.jpg
INFO  Generated: img/robot/qidai.jpeg
INFO  Generated: img/robot/python.png
INFO  Generated: img/robot/tuling.png
INFO  Generated: img/canteen/B.png
INFO  Generated: img/robot/dalao.png
INFO  Generated: lib/velocity/velocity.ui.min.js
INFO  Generated: lib/font-awesome/fonts/fontawesome-webfont.woff2
INFO  Generated: img/robot/result3.jpeg
INFO  Generated: img/canteen/F-319.png
INFO  Generated: img/canteen/F-320.png
INFO  Generated: img/robot/result1.jpeg
INFO  Generated: img/work/9.JPG
INFO  Generated: img/canteen/wen.png
INFO  Generated: lib/font-awesome/fonts/fontawesome-webfont.woff
INFO  Generated: img/canteen/E1.png
INFO  Generated: lib/velocity/velocity.ui.js
INFO  Generated: lib/velocity/velocity.min.js
INFO  Generated: img/canteen/E-322.png
INFO  Generated: img/canteen/F-321.png
INFO  Generated: img/canteen/F-322.png
INFO  Generated: img/canteen/F-323.png
INFO  Generated: img/canteen/Y-320.png
INFO  Generated: img/canteen/Y-321.png
INFO  Generated: img/canteen/Y-322.png
INFO  Generated: img/work/12.JPG
INFO  Generated: img/robot/result2.jpeg
INFO  Generated: img/work/4.JPG
INFO  Generated: img/canteen/E-319.png
INFO  Generated: js/src/algolia-search.js
INFO  Generated: lib/font-awesome/css/font-awesome.css.map
INFO  Generated: img/issue.png
INFO  Generated: img/combination4.png
INFO  Generated: img/canteen/E-323.png
INFO  Generated: img/work/3.JPG
INFO  Generated: img/canteen/Y-319.png
INFO  Generated: img/canteen/table.png
INFO  Generated: img/work/11.JPG
INFO  Generated: img/work/15.JPG
INFO  Generated: js/src/affix.js
INFO  Generated: js/src/exturl.js
INFO  Generated: js/src/next-boot.js
INFO  Generated: js/src/scrollspy.js
INFO  Generated: js/src/js.cookie.js
INFO  Generated: js/src/post-details.js
INFO  Generated: js/src/scroll-cookie.js
INFO  Generated: lib/font-awesome/fonts/fontawesome-webfont.eot
INFO  Generated: img/canteen/Y-323.png
INFO  Generated: img/canteen/E-321.png
INFO  Generated: img/canteen/E-320.png
INFO  Generated: lib/font-awesome/css/font-awesome.min.css
INFO  Generated: lib/font-awesome/css/font-awesome.css
INFO  Generated: js/src/motion.js
INFO  Generated: js/src/utils.js
INFO  Generated: js/src/schemes/pisces.js
INFO  Generated: img/work/16.JPG
INFO  Generated: js/src/schemes/muse.js
INFO  Generated: img/work/14.JPG
INFO  Generated: lib/jquery/index.js
INFO  Generated: img/install-steps.gif
INFO  Generated: lib/velocity/velocity.js
INFO  Generated: img/work/17.JPG
INFO  Generated: img/work/18.jpg
INFO  152 files generated in 1.86 s
feiy-mac:fainyang.github.io feiy$ hexo deploy     #部署到 GitHub
INFO  Deploying: git
INFO  Clearing .deploy_git folder...
INFO  Copying files from public folder...
INFO  Copying files from extend dirs...
[master f55c4d1] new hexo_blog
 34 files changed, 8859 insertions(+), 150 deletions(-)
 create mode 100644 archives/2019/03/index.html
 create mode 100644 archives/2019/index.html
 create mode 100644 categories/index.html
 create mode 100644 categories/学习/index.html
 create mode 100644 categories/生活/index.html
 create mode 100644 img/2019/github.png
 create mode 100644 post/hexo-blog/index.html
 create mode 100644 tags/NUS/index.html
 create mode 100644 tags/hexo/index.html
 rename tags/{life => links}/index.html (87%)
 create mode 100644 tags/工作/index.html
Enumerating objects: 114, done.
Counting objects: 100% (114/114), done.
Delta compression using up to 4 threads
Compressing objects: 100% (43/43), done.
Writing objects: 100% (69/69), 97.73 KiB | 3.62 MiB/s, done.
Total 69 (delta 33), reused 0 (delta 0)
remote: Resolving deltas: 100% (33/33), completed with 17 local objects.
To https://github.com/fainyang/fainyang.github.io.git
   2b7a9ca..f55c4d1  HEAD -> master
Branch 'master' set up to track remote branch 'master' from 'https://github.com/fainyang/fainyang.github.io.git'.
INFO  Deploy done: git
```
在浏览器输入: yourname.github.io 如果看到博客页面那就大功告成。如果有错误的话，在 GitHub repository Setting 里的 GitHub Pages 可以看到错误信息。  

![publish](/img/2019/publish.png) 

## 图片管理
[奥哥](https://wsgzao.github.io/)推荐的一个非常好用的图片管理工具 [PicGo](https://github.com/Molunerfinn/PicGo) 图片上传+管理新体验。

## 踩过的一些坑
1. 在添加标签和分类的时候，当在source下面创建了文件夹的 index.md 这个文件一定要添加相关的 type. 例如 type: "categories", type: "tags".
2. 添加[搜索功能](https://theme-next.iissnan.com/third-party-services.html#local-search)时，一直转圈加载不出来。是因为自己写的文章中存在不可见字符：一个backspace的字符。 详情可以参考： [一](http://luffy.studio/2018/04/11/Hexo%E4%B8%ADLocalSearch%E5%8D%A1%E6%AD%BB%E9%97%AE%E9%A2%98/)&nbsp; [二](https://www.sqlsec.com/2017/12/hexosearch.html)