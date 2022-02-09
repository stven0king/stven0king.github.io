---
title: 解决6.5.16及以上版本微信内部M页不能唤起APP
date: 2019-02-21 21:05:43
tags: [weixin]
categories: Android
description:  "为提升微信 webivew 中网页打开其他应用的体验，防止诱导点击、强制跳出等不合理行为， 我们的“唤起外部客户端”的能力统一调整为:在 6.5.16 及以上版本的微信客户端中，贵方网页将只能使用 launchApplication 接口，打开其他应用。该接口会在唤起前要求用户接受弹窗确认; 在 6.5.16 以下版本的微信客户端中，贵方网页可以继续使用现有方式，打开其他应用。
"
---

# [个人博客地址 http://dandanlove.com/](http://dandanlove.com/)

# 背景

<center>![深夜放毒](https://upload-images.jianshu.io/upload_images/1319879-d96e81fff0ce1659.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)</center>


最近微信唤起app的数据急速下降，产品同学告诉我们大事来了，微信不能唤起Android的App了~！~！

# 微信语录

为提升微信 webivew 中网页打开其他应用的体验，防止诱导点击、强制跳出等不合理行为， 我们的“唤起外部客户端”的能力统一调整为:
>- 1、 在 6.5.16 及以上版本的微信客户端中，贵方网页将只能使用 launchApplication 接口，打
开其他应用。该接口会在唤起前要求用户接受弹窗确认。
>- 2、 在 6.5.16 以下版本的微信客户端中，贵方网页可以继续使用现有方式，打开其他应用。


# 解决版本

我们在接入微信的 `opensdk` 的时候会在自己项目代码中包含 `xxx.xxx.xxx.wxapi.WXEntryActivity` 这个页面。
在 6.5.16 及以上版本的微信客户端中，微信首先唤起的是 `xxx.xxx.xxx.wxapi.WXEntryActivity` 这个页面，将参数放在 `extInfo` 字段中，由第三方 APP 自行解析处理 ShowMessageFromWX.Req 的微信回调。

```java
public class WXEntryActivity extends WXCallBack {
    @Override
    public void onReq(BaseReq req) {
        super.onReq(req);
        if (req != null && req instanceof ShowMessageFromWX.Req) {
            ShowMessageFromWX.Req request = (ShowMessageFromWX.Req) req;
            if (request.message != null && request.message.mediaObject != null
                    && request.message.mediaObject instanceof WXAppExtendObject) {
                WXAppExtendObject appExtendObject = (WXAppExtendObject) request.message.mediaObject;
                //唤起app的启动页面，将scheme协议中的数据进行透传
                Intent intent = new Intent(this, LaunchActivity.class);
                intent.setData(Uri.parse(appExtendObject.extInfo));
                startActivity(intent);
            }
        }
    }
}
```

微信官方具体描述我们可以参见： [微信webview唤起外部客户端接入说明2018版](https://download.csdn.net/download/stven_king/10969400)

# 总结
微信这样做，将微信与其下游的app的之前的影响继续加强。虽然我们做了不同的适配，但是同时能得到微信唤起app的成功或者失败的数据。在互联网产品竞争激烈的今天我们不仅仅要做好用户产品也好做好技术产品。



文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！