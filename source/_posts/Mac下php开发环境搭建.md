---
title: Mac下php开发环境搭建
date: 2016-08-12 11:23:05
tags:
---
>最近公司项目比较清闲，闲暇时间想学习学习php开发，毕竟不能吊死在一棵树上再加上最近iOS行情不太好，记录一下php的学习历程。
mac上php开发环境有两种配置方法，一种是自己一步步手动配置Apache+php+MySql+ phpMyAdmin，另外一种方法是直接下载xampp一键安装，直接集成了所有的开发环境，很便捷暴力。这里主要是记录一下前者的集成步骤。

<!--more-->
#### 一、 启动Apache
Mac系统已经集成了Apache环境，我们只需要一行指令就可开启Apache服务。
终端输入
`sudo apachectl start`
输入电脑密码，即可开启阿帕奇
`sudo apachectl -v`
可以查看版本信息
```
Server version: Apache/2.4.18 (Unix)
Server built:   Feb 20 2016 20:03:19
```
此时在浏览器输入`http://localhost`，会出现It works! ，就说明阿帕奇开启成功。
#### 二、 运行php
1. 找到Apache的配置文件，终端输入
`open /etc/apache2/`
找到http.conf文件，用文本编辑器打开，搜索 libexec/apache2/libphp5.so，如图

![QQ20160811-0@2x.png](http://upload-images.jianshu.io/upload_images/1642800-ab41b819794d7c8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
把搜索结果这一行前面的#去掉，保存文件。如果这里提示没有权限的话，选中该文件，右击点击显示简介，点击右下角的小锁解锁，然后点击左边的加号，把自己的账户添加进去并把权限设置为读与写，如果还是不行的话，就一层一层的往上修改文件夹的权限，知道可以修改为止。
2. 重启Apache
`sudo apachectl restart`
3. 替换网页文件
`open /Library/WebServer/Documents/`
复制index.html.en文件并重命名为info.php
3. 打开info.php ，输入你想输入的php代码再次重启Apache，在浏览器输入http://localhost/info.php，就可以看到代码运行结果了。

#### 三、 配置MySql
1. 去http://dev.mysql.com/downloads/mysql/ 下载对应的安装包。我选择的是
![QQ20160811-1@2x.png](http://upload-images.jianshu.io/upload_images/1642800-e26aee23f72bc062.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下载完成后进行安装，安装过程中会提示一个临时密码，要暂存一下，如图
![712523-232499c71bb79fcb.png](http://upload-images.jianshu.io/upload_images/1642800-a1ec806448bff3b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
安装完成后可以在Mac系统中选择启动或者关闭MySql。
2. 将MySql加入系统环境变量
```
(1).进入/usr/local/mysql/bin,查看此目录下是否有mysql
(2).执行vim ~/.bash_profile
    在该文件中添加mysql/bin的目录，如下图：
    PATH=$PATH:/usr/local/mysql/bin
添加完成后，按esc，然后输入wq保存。
最后在命令行输入source ~/.bash_profile
```
![712523-42aa5aabaeb7b2bb.png](http://upload-images.jianshu.io/upload_images/1642800-a26ebb79a6d6265a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 修改mysql密码，先登录MySql
`mysql -uroot -p`
输入的密码就是刚才暂存的临时密码
登陆成功后，通过下面命令修改密码
`SET PASSWORD FOR 'root'@'localhost' = PASSWORD('newpass');`

#### 四、使用phpMyAdmin
1. 下载phpMyAdmin
https://www.phpmyadmin.net/downloads/
2. 将下载好的文件解压，放进`/Library/WebServer/Documents/`文件夹中并命名为phpmyadmin
3. 复制`/Library/WebServer/Documents/phpmyadmin`中的`config.sample.inc.php`，并命名为`config.inc.php`，并放在当前文件夹下。
4. 编辑config.inc.php，修改其中的
`$cfg['Servers']]$i]['host'] = 'localhost';`
为
`$cfg['Servers']]$i]['host'] = '127.0.0.1';`即可
5. 在浏览器输入http://localhost/phpmyadmin ,输入用户名root和你刚才设置的密码，即可登录。

到此php的开发环境就配置完成了，如果觉得太麻烦了话，还是直接下载xampp吧，非常强大！

参考: http://my.oschina.net/joanfen/blog/171109
        http://www.jianshu.com/p/fd3aae701db9