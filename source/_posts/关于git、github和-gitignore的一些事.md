layout: pos
title: 关于git、github和.gitignore的一些事
date: 2016-05-17 17:06:19
tags:
---
# 给已有的本地项目添加git仓库并推送到远程仓库
<!--more-->
1. 创建git配置文件
cd到项目目录下，运行指令：$ git init
完成以后可以在项目所在文件夹中看到一个.git文件，默认是隐藏的
2. 把当前所有的文件添加到本地git库中
运行指令：$ git add . (注意空格)
3. 将add的文件commit
运行指令：$ git commit -m "注释语句"
4. 关联本地仓库到远程仓库（github、bitbucket等）
运行指令：$ git remote add origin https://github.com/xxx/xxxx
最后的链接是远程仓库的url地址
5. 上传代码到远程仓库
运行指令 ：$ git push -u origin master (push前最好git pull一下)

# 给git仓库添加.gitignore文件
当我上传一个demo到github时遇到一个问题，其中一个SDK大小超过了github单文件100M的限制，由于SDK是用cocoapods导入的，所以想是否可以忽略这些文件不做上传，经大神点播知道可以通过.gitignore实现。
.gitignore 顾名思义，用来设置忽略更改的文件，即不需要上传到git服务器的文件,例如令我们厌烦的xcuserstate的改变也可以通过这个方法忽略。
创建步骤：
在**根目录**创建.gitignore右击项目target，新建文件，右击target能保证是在根目录创建文件，如果不是根目录那么.gitignore将不会生效
![1.png](http://upload-images.jianshu.io/upload_images/1642800-ef4b486c7c16046b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
选择iOS->Other->Empety文件
![2.png](http://upload-images.jianshu.io/upload_images/1642800-a610e2e56e577ee7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
给文件命名为.gitignore
![3.png](http://upload-images.jianshu.io/upload_images/1642800-39bd964e766657dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
确认
![4.png](http://upload-images.jianshu.io/upload_images/1642800-76e2e5d84a45f9ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在.gitignore文件中添加常用的文件描述
![5.png](http://upload-images.jianshu.io/upload_images/1642800-5aa59243376a478c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
到这一步一般就完成了.gitignore文件的创建，如果文件不起作用的话，有可能是和缓存相关，可以尝试参照[这篇文章](http://stackoverflow.com/questions/11451535/gitignore-not-working)
另外附上OC常用的.gitignore描述，只需要把这段复制到.gitignore文件中并根据描述和自己需求选择需要忽略哪些类型的文件即可，来源：https://github.com/github/gitignore
```
# Xcode
#
# gitignore contributors: remember to update Global/Xcode.gitignore, Objective-C.gitignore & Swift.gitignore

## Build generated
build/
DerivedData

## Various settings
*.pbxuser
!default.pbxuser
*.mode1v3
!default.mode1v3
*.mode2v3
!default.mode2v3
*.perspectivev3
!default.perspectivev3
xcuserdata

## Other
*.xccheckout
*.moved-aside
*.xcuserstate
*.xcscmblueprint
*.xcscheme

## Obj-C/Swift specific
*.hmap
*.ipa

# CocoaPods
#
# We recommend against adding the Pods directory to your .gitignore. However
# you should judge for yourself, the pros and cons are mentioned at:
# https://guides.cocoapods.org/using/using-cocoapods.html#should-i-check-the-pods-directory-into-source-control
#
Pods/

# Carthage
#
# Add this line if you want to avoid checking in source code from Carthage dependencies.
# Carthage/Checkouts

Carthage/Build

# fastlane
#
# It is recommended to not store the screenshots in the git repo. Instead, use fastlane to re-generate the
# screenshots whenever they are needed.
# For more information about the recommended setup visit:
# https://github.com/fastlane/fastlane/blob/master/docs/Gitignore.md

fastlane/report.xml
fastlane/screenshots
```