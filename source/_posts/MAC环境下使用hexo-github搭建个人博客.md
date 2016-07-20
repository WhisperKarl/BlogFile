---
title: MAC环境下使用hexo+github搭建个人博客
date: 2016-07-20 17:15:09
tags:
---
hexo是一个快速、简洁且高效的博客框架，深受开发者青睐，在这里记录一下如何快速的用hexo+github搭建一个属于自己的个人博客。
### 初始化hexo
1. 安装node.js
可以到node.js的[官网](https://nodejs.org)根据自己的系统下载相应的安装包，Mac用户可以直接到我[网盘](http://yun.baidu.com/s/1hs4mZVu)下载，速度比较快
2. 安装git
mac下安装Xcode就自带git,相信这个对于程序员来说不是难事
3. 注册github账号
注册过程就不多说了，注册完成后需要新建一个仓库，并且命名为【username.github.io】，例如我的github用户名是WhisperKarl，所以我的仓库名是【[WhisperKarl.github.io](https://whisperkarl.github.io)】，同时这个也是把博客部署到github上以后你的博客的域名。
4. 初始化hexo
```
npm install -g hexo
```
安装完成后新建一个hexo文件夹，并cd到这个文件夹目录下执行
```
hexo init
npm install hexo --save
```
5. 本地预览
到第四步我们就完成了hexo的初始化操作，我们可以执行以下操作进行本地预览
```
hexo g (或 hexo generate)
hexo s (或 hexo server)
```
这时候应该会提示
```
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
```
这样我们就可以通过访问 http://localhost:4000/ 进行预览了
注意：如果在`hexo s`操作报错的话，可以运行下面命令后再重新尝试
```
npm install hexo-server --save
```

### 部署到github上
1. 打开创建的hexo目录，会有一个`_config.yml`文件，我们可以用`vim _config.yml`打开，也可以用其它编辑器（如sublime）直接打开修改。在_config文件中找到最下面，有deploy属性，修改为下面的样子：
```
deploy:
type: git
repository: https://github.com/WhisperKarl/WhisperKarl.github.io
branch: master
```
repository对应的就是你刚刚创建的github仓库的地址，注意前面有个空格。
2. 运行下面命令进行部署
```
npm install hexo-deployer-git --save
hexo g
hexo d (或 hexo deploy)
```
这样就可以在https://username.github.io 上看到自己的博客了。

### 博客的基本配置和发布新文章
1. 一些基本配置可以在`_config.yml`中进行修改，如博客名、作者名等，另外如果你采用了别的主题，在主题文件夹中也会有一个`_config.yml`文件，可以在里面进行主题的一些修改。
2. 发布新文章时，先执行以下命令：
`hexo new post '文章标题'`
这时在source->_posts文件下能看到新建的文章，是md格式，hexo支持markdown语法，我们可以用markdown编辑器进行编辑。
3. 更新到github
我们每次做了修改后，都需要执行以下命令以更新github配置
```
hexo clean
hexo g
hexo d
```

### 如何在多台电脑上管理hexo博客
这时候我们一样可以利用伟大的github，我们可以在github建一个仓库用来管理hexo文件目录，这样我们的每个操作都可以推到git上方便我们多台电脑同时管理。需要注意的是如果我们的主题是从github上拉下来的，那我们在推到自己的github上时需要移除主题的git仓库，否则主题是不会被上传到github上的，此外，当我们在一台新的电脑上拉取hexo文件后，需要重新配置hexo环境。
