---
layout: post
title:  "微信小程序跳转App时，app-parameter信息的传递研究"
date:   2019-03-26 16:28:29 +0800
categories: iOS
---

## 问题描述

微信的小程序在跳转App时，是可用通过`app-parameter`来传递一些额外信息

```js
<button open-type="launchApp" app-parameter="additionalInfoxxxxx" binderror="launchAppError">打开APP</button>
```

但我们在 `AppDelegate` 中的 `- (BOOL)application:openURL:sourceApplication:annotation:` 中打断点后，可以看到，只有没什么信息量的url和为nil的annotation：

![-w406](/media//15535935678385.jpg)

url的scheme为App注册的固定的微信scheme，annotation为nil，可见没有任何其他消息

接下来，我们按照微信的文档来实现获取参数：

```objc
BOOL result = [WXApi handleOpenURL:url delegate:[WechatSDKDelegate sharedInstance]];
```

```objc
// WechatSDKDelegate

- (void)onReq:(BaseReq*)req
{
    if ([req isKindOfClass:[LaunchFromWXReq class]]) {
        LaunchFromWXReq *launchReq = req;
        NSString *appParameter = launchReq.message.messageExt;
        // do sth bellow
    }
}
```

可以看到，在 `onReq:` 竟然能取到结构很复杂的 `req`，而 `req.message.messageExt` 就是我们从小程序中传过来的 `app-parameter`。那么问题来了，我们在上面可以看到，除了一个很简短的url，没有任何其他信息了，这个 `messageExt` 的内容是从哪里获取的呢？

## 猜测

经过和同事讨论，猜测了几种可能性：

1. 可能是微信SDK发了个网络请求来处理这个事情。但抓包后并没有
2. 微信向目标App中的进程做了什么事情。但iOS的保护机制应该没那么容易做到，起码目前没想到可以绕开权限的
3. App间进行socket通信，但考虑到需要起服务、保活等因素感觉也不是太可能

## 调查

没办法了，只能去调查下SDK的静态库了，用Hopper打开后，看了几个类的方法后，终于找到了一个方法感觉很像，并且给人豁然开朗的感觉。

![-w1679](/media//15535958910573.jpg)

就是这个 `[WXOMTAOpenUDID _getDictFromPasteboard:]` 方法！

有意思，猜测可以通过剪切板将需要的信息传递过来，在App中获取后再做好处理就OK了。那么，接下来就是验证了。

我将 `app-parameter` 中线传数据A两次，然后传一次数据B，最后再传一次A，然后我在App中可以从剪切板中获取到以下的数据：

![-w914](/media//15535967774203.jpg)


可以看出，第 1、2、4的剪切板数据是一样的，第3次不一样。这里就可以猜测出来的确是用的剪切板了😆

下面将数据转换一下：

![-w549](/media//15535965424547.jpg)

被圈出来的地方就是我之前小程序中 `app-parameter` 中传过来的值。

OK，破案。