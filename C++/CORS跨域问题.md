## 什么是同源策略？
如果两个URL的协议、域名和端口相同的话，则说明这两个URL是同源的。同源策略是浏览器通过的一种安全策略，它用于限制一个源的文档或者它加载的脚本如何能与另一个源的资源进行交互。简单来说，如果网站A和网站B不同源，那么网站A就不能读取非同源网页B的Cookie、localStorage，也不能接触B网站的DOM。而且A无法向非同源地址发送AJAX请求。同源策略能帮助阻隔恶意文档，减少可能被攻击的媒介。

浏览器允许发起跨域请求，但是跨域请求回来的数据会被浏览器拦截，Web页面无法获取。

## CORS是什么？
CORS跨域资源共享是一个W3C标准，它是一种基于HTTP头的机制，该机制通过允许服务器标示除了它自己以外的其他源，这样浏览器就可以向跨域的服务器发送AJAX请求访问加载资源了。

## 怎么实现CORS？
CORS需要浏览器和服务器同时支持，比如IE浏览器需要IE10以上，浏览器一旦发现AJAX请求跨域，就会自动添加一些附加的头信息，有时还会多出一次附加的请求。实现CORS通信的关键是服务器。只要服务器实现了CORS接口，就可以跨域通信。

## 什么是预检请求（preflight request）？预检请求有什么作用？
（HTTP 的 OPTIONS 方法 用于获取目的资源所支持的通信选项。）

预检请求是用于检查服务器是否支持跨域资源共享，浏览器使用 OPTIONS 方法发起一个预检请求。它一般是用了以下几个 HTTP 请求首部的 OPTIONS 请求：Access-Control-Request-Method 和 Access-Control-Request-Headers，以及一个 Origin 首部。

~~~
OPTIONS /resource/foo
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: origin, x-requested-with
Origin: https://foo.bar.org
~~~

如果服务器允许，那么服务器就会响应这个预检请求。并且其响应首部 Access-Control-Allow-Methods 会将 DELETE 包含在其中：

~~~xml
HTTP/1.1 200 OK
Content-Length: 0
Connection: keep-alive
Access-Control-Allow-Origin: https://foo.bar.org
Access-Control-Allow-Methods: POST, GET, OPTIONS, DELETE
Access-Control-Max-Age: 86400
如果服务器否定了“预检”请求，会返回一个正常的 HTTP 回应，Access-Control-Allow-Origin字段明确不包括发出请求的Origin。
~~~

## 简单请求是什么？
不会触发 CORS 预检请求的请求称为简单请求。如果它的请求方式是GET、HEAD或者POST请求，且HTTP头部字段不超出Accept、Accept-Language、Content-Language、Content-Type:(text/plain、multipart/form-data、application/x-www-form-urlencoded则为简单请求。

不满足条件的就属于非简单请求。

multipart/form-data：可用于HTML表单从浏览器发送信息给服务器。作为多部分文档格式，它由边界线（一个由’–'开始的字符串）划分出的不同部分组成。每一部分有自己的实体，以及自己的 HTTP 请求头。一般用于文件。

~~~
text/plain： 是使用纯文本进行传输，平时用的很少

application/x-www-form-urlencoded ：数据被编码成以 '&' 分隔的键-值对, 同时以 '=' 分隔键和值。

举例：向服务器发送数据 {a:"a",  b:"b"}

如果头的格式是application/x-www-form-urlencoded,  则ajax.send("a='a'&b='b'");

如果头的格式是application/json,  则ajax.send(JSON.stringify(data));
MIME 类型 - HTTP | MDN (mozilla.org)
~~~

### 简单请求的流程？
浏览器发现这次跨域 AJAX 请求是简单请求，就自动在头信息之中，添加一个Origin字段。如果Origin指定的源，不在许可范围内，服务器会返回一个正常的 HTTP 回应。浏览器发现，这个回应的头信息没有包含Access-Control-Allow-Origin字段，就知道出错了，从而抛出一个错误，被XMLHttpRequest的onerror回调函数捕获。注意，这种错误无法通过状态码识别，因为 HTTP 回应的状态码有可能是200。

## CORS响应头有几种？作用是什么？

**Access-Control-Allow-Origin**：指定了允许访问该资源的外域 URI，通配符*表示允许来自所有域的请求。

**Access-Control-Allow-Methods**: 用于预检请求的响应，其指明了实际请求所允许使用的 HTTP 方法。

**Access-Control-Allow-Headers**：用于预检请求的响应。其指明了实际请求中允许携带的首部字段。

**Access-Control-Allow-Credentials**：响应头表示是否可以将对请求的响应暴露给页面。返回true则可以，其他值均不可以。当作为对预检请求的响应的一部分时，这能表示是否真正的请求可以使用credentials。

**Access-Control-Expose-Headers**：在跨源访问时，XMLHttpRequest 对象的 getResponseHeader() 方法只能拿到一些最基本的响应头，比如Content-Language、Content-Type等，如果要访问其他头，则需要服务器设置本响应头。

**Access-Control-Max-Age**：预检请求的结果能被缓存多久，即在多少秒内有效。

## CORS请求头有几种？作用是什么？
**Origin**：表明预检请求或实际请求的源站，值为源站 URI，它不包含任何路径信息，只是服务器名称。

**Access-Control-Request-Method**：用于预检请求。其作用是将实际请求所使用的 HTTP 方法告诉服务器。

**Access-Control-Request-Headers**：用于预检请求。其作用是将实际请求所携带的首部字段告诉服务器。

## CORS的优缺点是什么？

CORS是 W3C 标准，属于跨域 Ajax 请求的根本解决方案，支持 GET 以外的其他请求，比如 POST 请求。缺点是出现的晚，不兼容某些低版本的浏览器，比如IE浏览器需要IE10以上。