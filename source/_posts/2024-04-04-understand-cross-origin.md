---
layout:     post
title:      理解前端跨源问题
subtitle:  
date:       2024-04-04
author:     "Chance"
catalog:  true
tags:
    - 前端
---

# 什么是跨源请求？

先看看什么是同源 URL：

> 如果两个 URL 的[协议](https://developer.mozilla.org/zh-CN/docs/Glossary/Protocol)、[端口](https://developer.mozilla.org/zh-CN/docs/Glossary/Port)（如果有指定的话）和[主机](https://developer.mozilla.org/zh-CN/docs/Glossary/Host)都相同的话，则这两个 URL 是同源的。这个方案也被称为“协议/主机/端口元组”，或者直接是“元组”。（“元组”是指一组项目构成的整体，具有双重/三重/四重/五重等通用形式。）

当网站的 URL 和网站发出请求的 URL 是非同源的，我们便说这个请求是跨源请求。

# 请求为什么不能跨源？

假如 A 是银行网站，你在 A 网站上进行了登录，A 网站将 token 保存在浏览器的 cookies 中。接下来你收到了一封钓鱼网站 B 发来的邮件，并点开了其中的连接，然后 B 网站在你不知情的情况下调用 A 网站的转账接口，将你在这家银行的钱全转进他的帐户里。由于转账接口的域名是 A 网站的，因此这个请求会把浏览器保存的 A 网站 cookies 也一并带上去，于是该请求就能顺利通过 A 网站服务器的身份验证，然后你💰就没了 : (。

<!--more-->

# 如何解决跨源请求带来的安全问题？

一开始，浏览器有下面几种机制来避免上述情形。

第一种机制是，不允许 B 网站发送任何 A 网站域名下的 Ajax 请求，发送的话直接报错。

第二种机制是，对于 `img`​、`script`​ 、`link`​ 这类标签触发的跨源请求和响应，加以限制。我们知道，浏览器是允许 `img`​、`script`​、`link`​ 这类标签触发跨源的请求的。也就是说， B 网站可以通过 `img`​ 和 `script`​ 标签来获取 A 域名下的图片和脚本，而且允许带上 A 网站的 cookies。之所以这样做，大概是因为图片和脚本这类静态资源，经常需要托管到专门的服务器比如 CDN 上进行存储。对于正常网站来说，标签触发的跨源请求只是为了获取 CDN 资源，允许它们没什么问题，但恶意网站会利用这个特性去获取用户在其他网站的信息。比如，黑客可能会在自己的网页里嵌入 `<img>` 标签来获取用户在其他网站的图片，然后将该 `<img>`​ 先绘制在 canvas 上，再将 canvas 上的像素信息进行转储，从而间接获取到这张图片。

安全起见，浏览器对于标签触发的跨源请求做了一些限制。具体措施是，如果某张 `canvas`​ 绘制了跨源的 `<img>`​，这张 `canvas`​ 就被“[污染](https://developer.mozilla.org/zh-CN/docs/Web/HTML/CORS_enabled_image#%E5%AE%89%E5%85%A8%E6%80%A7%E5%92%8C%E2%80%9C%E8%A2%AB%E6%B1%A1%E6%9F%93%E2%80%9D%E7%9A%84_canvas)”了，后续便无法将 `canvas`​ 转储为图片文件（[`toBlob()`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLCanvasElement/toBlob)​）或者 data url （[`toDataUrl()`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLCanvasElement/toDataURL)​）导出，总之就是允许写，不允许读了。

除了 `img`​ ，浏览器对 `script`​ 等其他标签也做了类似的限制，这里就不展开讲了。

第三种机制是把 cookies 的作用域限制在本网站。B 网站能够成功窃取用户在 A 网站的信息，根源在于 cookies 的作用域是对请求域名的限制，而非网站域名。A 服务器给 A 网站下发 cookies 本意只想让 A 网站用，但 cookies 的默认语义是，只要是发给 A 域名服务器的请求，不管是哪个网站发送的，都会带上 A 网站的 cookies。如果将 cookies 的作用域限制在 A 网站上，自然就不存在这个问题了。由于这种方式要打破 cookies 的默认语义，因此服务器下发 cookies 的时候，需显式告知浏览器，只有和该 cookies 域名相同的网站才能在发送请求时把它附带上去。具体做法是，服务器在 cookies 中加上 [`SameSite`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Set-Cookie#samesitesamesite-value)​ 属性。

三种机制在不同场景下发挥作用，共同保护用户的隐私和财产。乍一看这一切都很完美，但对开发者来说，这些限制有时候会带来一些麻烦。考虑以下几种场景：

- 场景 1：通过 Ajax 获取自己在另一台服务器上托管的资源。
- 场景 2：获取自己控制的另一个域名下的 `<img>`​ 并将其绘制在 `canvas`​ 上，然后转储为图片。

由于一刀切政策，上述需求无法通过常规手段实现。为了绕过这种限制，程序员想了很多奇技淫巧，比较常见的解决方案是 JSONP。JSONP 利用的 `script`​ 可以跨源的特性，下面我们看看它是怎么玩的。

# 使用 JSONP 突破跨源限制

JSONP 把跨域 Ajax 请求伪装成一个外部 script 请求，这是网页和目标网站之间的“秘密”，浏览器并不知情，因此浏览器并不会阻拦，而是正常发送该请求以获取跨源脚本。重点来了，服务器返回的脚本并不是普通的静态脚本，而是根据请求动态生成的。这段脚本会以数据作为参数去执行发送请求时传给服务器的回调函数。这么说可能有点难理解，我们通过具体例子来说明。

网站 A (https://a.com) 想要发送 https://api.b.com/get-user-info 这样一个请求给服务器 B。因为跨源，所以无法发送 Ajax 请求。于是它动态地给网页添加了一个标签：

```javascript
function onGetUserInfoSuccess(data) {
  console.log(data);
}

function onGetUserInfoError(e) {
  console.log(e);
}

const script = document.createElement('script');
script.src = 'https://api.b.com/get-user-info?onsuccess=onGetUserInfoSuccess&onerror=onGetUserInfoError';
document.head.appendChild(script);
```

浏览器正常发送了这个外部脚本请求，服务器 B 收到请求后返回如下脚本：

```javascript
onGetUserInfoSuccess({"Name": "小明", "Id": 1823, "Rank": 7});
```

浏览器得到该脚本后会立即执行，而执行上述脚本的结果就是网站 A 获取到了来自服务器 B 的数据。实际 JSONP 的做法和这个例子有点差异，但原理是一样的。这种跨源请求方案神不知鬼不觉地绕过了浏览器的限制，这显然是在玩火，它是存在[安全隐患](https://zh.wikipedia.org/wiki/JSONP#%E5%AE%89%E5%85%A8%E5%95%8F%E9%A1%8C)的，需要目标服务器做好处理。

奇技淫巧也只是权宜之计，JSONP 并没有完全解决跨源限制的问题，开发者还是希望能浏览器能原生支持受信任的跨源请求。终于，千呼万唤之下，跨源资源共享（CORS）出现了。

# 迎接 CORS

跨源资源共享是对一刀切机制的改进。一刀切问题的根源在于，它默认所有网站都不被域名之外的服务器所信任。这种假设非常简单粗暴，没有考虑到：如果能得到目标服务器授权，请求即使跨源也应该是被允许的。防止敌人混入城堡的正确方式是在城门口配上守卫，对进来的人进行身份验证，而不是装上铁门后焊死。那么，CORS 是如何尽好守卫职责的呢？CORS 会在发送跨源 Ajax 请求之前，先发送一个所谓的预检请求给目标服务器，看看目标服务器是否允许该来自该网站的跨源请求，如果允许，跨源请求就正常发出；如果不允许或者不做出响应，就代表拒绝，浏览器则抛出异常。这个过程有点像是在 “请示” 上级的批准。

为什么服务器对预检请求不作出响应也视为失败？其实是为了兼容。引入 CORS 之前，很多服务器可能依赖于浏览器禁止跨源的一刀切政策来实现有限程度的安全，浏览器单方面引入 CORS 后，服务器的认知依然停留在 CORS 出现之前的时代，它们根本不会理会预检请求。如果浏览器把服务器的 “不理会” 理解成允许，擅自将实际请求发给它们，它们可能以为这个请求已经通过了浏览器的跨源访问限制，故而不再做进一步的验证，直接给出响应，这将会引发大面积的安全危机。

除了 Ajax，CORS 也对标签触发的跨源资源访问在一定程度上放开了限制：只要这个请求能够得到目标服务器的 ”批准”，就不再对响应数据进行访问限制。为了兼容性，要想突破这种限制，标签需要显式声明自己期望接受  CORS 的身份验证，具体做法就是给 `img` 标签加上 [`crossorigin`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/img#crossorigin)​属性。和 Ajax 一样，浏览器处理这类标签触发的请求时，会先发送一个预检请求，如果请求被服务器“批准”，网站就能突破对响应数据的访问限制。比如，使用了 `crossorigin`​ 属性的 `<img>`​ 标签，浏览器会在请求跨源图片时，给服务器发送预检请求，如果服务器拒绝，那么就抛出异常，否则就把图片加载进来。加载后图片可以随意使用，浏览器不再做限制。例如，客户端网页可以把它画在 `canvas`​ 上，然后将其转储为图片文件。

CORS 既没有降低安全性，又让受信任的跨源请求变得更加容易，算是比较好的解决方案。但没有什么是绝对完美的，CORS 也有瑕疵，[有一种特别隐秘的方式可以获取用户在某个网站上的登录状态](https://www.grepular.com/Abusing_HTTP_Status_Codes_to_Expose_Private_Information)，虽然该问题并不是 CORS 引入的，但应该是由 CORS 去解决的。

# 结语

我这里省略了很多 CORS 的具体玩法，如果感兴趣可以参考 MDN 上的相关[文档](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS)，文档针对 “怎么做” 讲得很清楚，我就不赘述了，这篇博客只是想补充一些背景供大家参考，也算是我自己的笔记。文中有些地方是我自己的理解，有不对的地方还请不吝赐教。

‍
