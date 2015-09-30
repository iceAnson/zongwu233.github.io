---
layout: post
title: 不规范的HTTP 204 响应格式
description: HTTP response 要严格遵循规范哦
category: blog
---


总悟君这次要说的是最近遇到的关于Http返回异常格式的问题。<br>
起因是这个:


```
SyncNetworkPerformer.execute: 
Unhandled exception com.zongwu.common.http.volley.NoConnectionError: java.net.ProtocolException: 
Unexpected status line: nullHTTP/1.1 200 OK   
com.zongwu.common.http.volley.NoConnectionError: java.net.ProtocolException: Unexpected status line: nullHTTP/1.1 200 OK           
at com.zongwu.common.http.volley.toolbox.BasicNetwork.performRequest(BasicNetwork.java:163)  
at com.zongwu.common.http.volley.SyncNetworkPerformer.execute(SyncNetworkPerformer.java:64)
```
什么情况，服务端居然返回给我错误的http头部？？但是现在还不能找服务端的童鞋~~撕x~~理论，因为还有几个疑点我还回答不上来。				
1. 该问题不是必现。					
2. 线上版本跑的好好的，我把HTTP请求库切换成volley就出问题了。				
总悟君很快就必现了bug：反复调用该方法，HTTP返回204之后的下一次一定出现该问题。那新问题又出现了，**HTTP是无状态的请求**，为何上一次的请求可以影响下一个请求呢？答案是TCP通道的复用。[html rfc2616#8.1.2](http://tools.ietf.org/html/rfc2616#section-8.1.2) 介绍了HTTP1.1默认开启持久化连接。
仔细查看线上版本的代码发现每一次Http请求完成后都调用了shutdown()方法。

```
DefaultHttpClient client = getHttpClient(httpPost.getURI().toString());
  ...
  client.getConnectionManager().shutdown();
```
这个当然是关闭了TCP通道的链接。
两个疑点都搞定，但是这个null到底是什么时候塞进来的呢？总悟君祭出大杀器**WireShark** 。(幽默风趣的[林沛满](http://weibo.com/linpeiman)为WireShark写了一本很有意思的入门[书籍](https://book.douban.com/subject/26268767)推荐读一读)
设置好wifi热点，手机连到该wifi上，开始抓包。
出现问题的包是这样的
![图片](http://7xn3gz.com1.z0.glb.clouddn.com/204-response-with-null.png)
2 reassembled TCP Segments 149 bytes?		
查看 \#44005 最后的4个bytes就是null。而这个\#44005是上一次的HTTP 204返回结果的TCP序号。\#44005的数据已经发送到了客户端，为何最后4个bytes再次被算进来了？？
让我们先去看看HTTP 1.1对204返回值的[规范](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html):		

>10.2.5 204 No Content

>The server has fulfilled the request but does not need to return an entity-body, and might want to return updated metainformation. The response MAY include new or updated metainformation in the form of entity-headers, which if present SHOULD be associated with the requested variant.

>If the client is a user agent, it SHOULD NOT change its document view from that which caused the request to be sent. This response is primarily intended to allow input for actions to take place without causing a change to the user agent's active document view, although any new or updated metainformation SHOULD be applied to the document currently in the user agent's active view.

>The 204 response **MUST NOT**include a message-body, and thus is always terminated by the first empty line after the header fields.				


**最后一段明确说明204响应不允许包含消息体。**

于是总悟君脑补了一下客户端和服务端的对话：

			
客户端：hi 你好 （SYN）			
服务端：hi 你好	（SYN）	
客户端：耶 联系上你拉	（ACK）
(三次握手，建立TCP通道)		
客户端：我想要查看某个资源 (GET /ad?xxx=x)		
服务端：收到你的请求了 （ACK）		
服务端：该资源没更新 
	（HTTP/1.1 204 NO Content  ... **null** ）	（**注意结尾带了个null**）	
客户端：收到（ACK）		
（客户端内心独白）：204啊那就没有任何数据需要consume咯。(刚才出现的**null**字符残留在tcp接收缓冲区内)		
（本次HTTP请求结束。但是TCP长链接没有断开）		
客户端：我想要查看某个资源 (GET /ad?xxx=x)			
服务端：收到请求，有数据下发（ACK）		
服务端：HTTP/1.1 200 OK		
客户端：（取数据） nullHTTP/1.1 200 OK 。。。 服务端！你给我了啥。。。 (读取接收缓存区 的时候 ，上次残留的null字符也被一起读进来，问题出现了)



问题原因找到。服务端童鞋按照规范修改了204的返回格式。一切都正常了~










