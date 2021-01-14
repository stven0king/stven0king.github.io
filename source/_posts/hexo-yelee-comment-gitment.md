---
title: Hexo博客yelee主题添加Gitment评论系统
date: 2017-10-26 19:40:00
tags: [gitment]
categories: 搞事情
description: "自从六月份多说评论关闭后，接着好不容易迁到网易云跟帖。8月1日网易云跟帖发布公告宣布停止服务。看到wordpress博客大部分接的是畅言，可惜畅言需要网址备案，没有买阿里云服务器域名不给备案。今天突然看到了gitment（PS:gitment是imsun利用github上的issues做的评论系统），相见恨晚啊。"
---
## 前言

自从六月份多说评论关闭后，接着好不容易迁到网易云跟帖。8月1日网易云跟帖发布公告宣布停止服务。看到wordpress博客大部分接的是畅言，可惜畅言需要网址备案，没有买阿里云服务器域名不给备案。今天突然看到了gitment（PS:gitment是imsun利用github上的issues做的评论系统），相见恨晚啊。然后自己立即着手开始自己的博客评论系统迁移，一个小时不到就搞定了^_^，成功的方法和喜悦给大家分享一下。

## 注册OAuth Application

[OAuth Application注册](https://github.com/settings/applications/new)

![Register](https://img-blog.csdnimg.cn/img_convert/036365a9ad16f8fe669a1f99d79f8edc.png#pic_center)

> 注意callback URL需要填自己的博客地址，eg:http://dandanlove.com/

创建成功后，你会得到一个 client ID 和一个 client secret，这个将被用于之后的用户登录

![密钥](https://img-blog.csdnimg.cn/img_convert/001a10da2e545f825ce5ec4762b81efe.png#pic_center)

## 在yelee主题中引入Gitment

在themes/yelee/layout/_partial/post文件夹下创建git.ejs文件，并写入下边代码：
```html
<!-- Gitment评论插件通用代码 -->
<div id="git"></div>
<link rel="stylesheet" href="https://imsun.github.io/gitment/style/default.css">
<script src="https://imsun.github.io/gitment/dist/gitment.browser.js"></script>
<script>
var gitment = new Gitment({
  owner: "stven0king",//github用户名
  repo: "stven0king.github.io",//用户存储评论的github项目名称
  oauth: {
    client_id: "xxxxxxxxxxxxxxxxxxxxxxxx",//注册OAuth Application时生产的ClinetID
    client_secret:"xxxxxxxxxxxxxxxxxxxxx",//注册OAuth Application时生成的Client Secret
  },
})
gitment.render('git')
</script>
<!-- Gitment代码结束 -->
```

接着在`themes/yelee/layout/_partial/article.ejs`文件中找到

```html
<% if (!index && post.comments){ %>
    <% if (theme.duoshuo.on) { %>
      <%- partial('comments/duoshuo', {
          key: post.path,
          title: post.title,
          url: config.url+url_for(post.path),
          }) %>
    <% } else if (theme.youyan.on) { %>
        <%- partial('comments/youyan') %>
    <% } else if (theme.disqus.on) { %>
        <%- partial('comments/disqus', {
            shortname: theme.disqus.shortname
          }) %>
    <% } else if (config.disqus_shortname) { %>
        <%- partial('comments/disqus', {
            shortname: config.disqus_shortname
          }) %>
    <% } %>
<% } %>
```
在这个节点下添加：
```html
<% if (!index){ %>
  <% if (post.comments){ %>
  <%- partial('post/git') %>
  <% } else { %>
    <div class="git"></div>
  <% } %>
<% } %>
```
以上所有操作完成后，文章底部就可以展现评论视图了。

## 初始化评论

最开始我们看到的是：Error:Comments Not Initialized

![Error:Comments Not Initialized](https://img-blog.csdnimg.cn/img_convert/2bba97b35081ebe3e6d542aa9007b8e7.png#pic_center)

出现Error之后我们不要惊慌，点击评论部分的Login。在GitHub进行授权后页面会刷新成：Initialize Comments

![Initialize Comments](https://img-blog.csdnimg.cn/img_convert/dfd1b365dd112217092e8583f846ce75.png#pic_center)

接着点击【Initialize Comments】按钮进行初始化就可以评论了。

![issues](https://img-blog.csdnimg.cn/img_convert/2b914cbc9665df9094c5e74240c8cf3d.png#pic_center)

没初始化一片文章都会在repo所在的github项目的issues中看到。

参考文章：
[Gitment](http://blog.csdn.net/anttu/article/details/77688004)

[gitment github项目地址](https://github.com/imsun)

[gitment 错误处理](https://github.com/imsun/gitment/issues)

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
![振兴书城](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktNjEyYzRjNjZkNDBjZTg1NS5qcGc#pic_center)