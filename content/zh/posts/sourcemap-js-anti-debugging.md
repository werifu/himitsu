---
title: SourceMap与JS反调试
description: trick? 用SourceMap来做前端反调试
toc: true
authors: werifu
tags: [前端, 安全, 浏览器]
categories: []
series: []
date: 2021-10-28T18:40:44+08:00
lastmod: 2021-10-28T18:40:44+08:00
featuredVideo:
featuredImage:
draft: false
---
这篇文是2021秋的一次联创web组内分享

## SourceMap是什么
发送到前端的代码往往不是写的时候的样子
![](https://s3.bmp.ovh/imgs/2022/05/28/81cc10f6f6c59a73.png)
为了方便调试，在SourceMap文件中规定了源文件和打包后的文件的映射。
![](https://s3.bmp.ovh/imgs/2022/05/28/482eac2708c424a3.png)
```javascript
// index.js
function foo() {
    return getNum();
}
// 只需要在改造后的代码最后加上这一行注释就会被解析
//# sourceMappingURL=http://example.com/index.map
```
```typescript
// index.ts
// ...
function foo():number {
    return getNum();
}
```
```json
// index.map(其实是一个json文件
// 正儿八经的SourceMap并不这么写
{
    "sources": ["path/index.ts"],    // 可以有很多个
    "mappings":
        "index.js第二行第一句映射到index.ts的第三行第一句,
        index.js第x行第y句映射到index.ts第i行第j句,
        CAAEA"
}
```

> MDN的SourceMap使用demo
https://mdn.github.io/devtools-examples/sourcemaps-in-console/index.html

## SourceMap的一些feature
1. 只有在DevTools中打开enable source map才有效，因此注释里的链接是可以动态构造的
2. 打开DevTools时才会加载Map
3. 由于可以使用网络来获取SourceMap，因此一定可以发送网络请求(GET)
4. 网络请求不会在DevTools的Network中展示（当然wireshark等是能抓到的）
5. 传输是单向的（即不会解析response）
## SourceMap的非正经用法
### 悄悄发送请求

之前说可以动态构造链接，那么只要由脚本控制，可以随时发送get请求出去，而且不会留下明显的痕迹。（可以随时remove
```javascript
const report = (url, data) => {
    const script = document.createElement('script');
    script.textContent = `//# sourceMappingURL=${url}?data=${data}`;
    document.head.appendChild(script);
    script.remove();
}
report('/report', 'value');
```

### 监听DevTools的打开事件

- 之前说没法解析response？如何做到双向沟通？
有没有办法不用解析response也能将状态保留到本地的方式？
Set-Cookie && document.cookie
(我们认为网站的前端是😈，因此httpOnly的header由😈把控，可以设置为false，这样js就能够获取该cookie）
- 监听这个能干嘛？
比如你有怪浪怪浪的😈代码，在调试人员打开devtools的时候就可以删除该部分代码或者修改成人畜无害的👼代码，这样就达到了Anti-debugging的目的。

比如最极端的你可以
`document.body.innerHTML = "";`

鉴于其操作的高灵活性，即使不是做坏事也可以玩出一些花来，可以用于跟用户开玩笑。
### 绕过内容安全策略(Content Security Policy)
内容安全策略是浏览器用于保护服务器的一种安全限制，可以由服务端设置安全策略，可以防止许多XSS的注入。

Eg:
服务端返回时带上了以下header，只允许来自向同源域名发送请求（或者说这个页面下的所有资源都应当来自同源域名），不符合要求的请求将会被浏览器拦截
Content-Security-Policy: default-src 'self'

但是用sourceMappingUrl可以绕过这个限制，就是说我们可以
`sourceMappingUrl = http://others.com` 而不受影响

## Sum up
一个有意思的trick，提供了一种在前端做小动作而不会在DevTools里暴露的方法，比如：
- 悄悄地发送一些请求
- 在有开发人员打开调试界面的时候可以将以前的手脚删除以达到Anti-Debugging的效果，在调试过程中会发现整个代码人畜无害。
- 用来实现【监听DevTools启动】这样一个浏览器中没有提供的事件。

不过缺点也很明显：
- 毕竟报文发送出去了，wireshark/sniffers都是能抓到包的，同时sourcemap相关代码也无法完全隐藏起来，无法完全做到隐身
- 需要浏览器打开SourceMap功能才能正常工作（不过似乎现在浏览器都默认打开的？未证实
- DevTools打开时才能够加载SourceMap，如果是想做监控显然不够全面，因为绝大多数人打开网页后不会去按F12
因此还是作为一个JavaScript Anti-Debugging的一个trick，而不是无敌的存在。


## 参考资料
- 主要的内容来源于这一篇blog：
https://weizman.github.io/?javascript-anti-debugging-some-next-level-shit-part-1
- [JavaScript Source Map 详解 - 阮一峰的网络日志](https://developer.mozilla.org/zh-CN/docs/Tools/Debugger/How_to/Use_a_source_map)
- [使用 source map - Firefox 开发者工具 | MDN](https://developer.mozilla.org/zh-CN/docs/Tools/Debugger/How_to/Use_a_source_map)
- [内容安全策略( CSP ) - HTTP | MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP)
- [微型demo](https://github.com/werifu/sourcemap-js-anti-debug)


