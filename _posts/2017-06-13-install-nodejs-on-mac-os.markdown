---
layout:     post
title:      "Install Node.js on Mac OS"
#subtitle:   ""
date:       2017-06-13 15:49:39
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Node.js
    - MacOS
---
在[Ebates.cn](https://www.ebates.cn/)的时候搞过一段[Node.js](https://nodejs.org/en/)，可惜商城项目无疾而终，所以也没能坚持下来。

正好想改改Blog所用[Theme](https://github.com/Huxpro/huxpro.github.io)的样式，所以再重温下最基础的安装流程。

其实只是想改下\<p\>标签的margin top的值，30px感觉有点儿空旷，20px感觉不错。看了下目录结构，应该是改改less目录下的hux-blog.less文件就行了。这里也可以一步到位，直接把文件名改成自己的。

用Mac就是爽，手动微笑，因为有[Homebrew](https://brew.sh/)这个神器。

1. `brew install node`
2. `npm install -g grunt-cli`  
    安装好了[Grunt](https://gruntjs.com)，别急着在项目目录下执行`grunt`命令噢，现在还没好呢~
3. 修改`package.json`文件中的相关内容，其中name改成echo-blog
4. 重命名less目录下的`hux-blog.less`为`echo-blog.less`，重命名js目录下的`hux-blog.js`为`echo-blog.js`。  
   得益于Git的特性，它能识别出来这是Rename操作。
5. `npm install`
6. `grunt`  
   能看到有less的expanded和minified操作。

就酱紫~