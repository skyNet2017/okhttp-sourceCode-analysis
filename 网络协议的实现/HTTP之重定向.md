# http协议中的重定向
> client:
> 向server发送一个请求，要求获取一个资源
> server:
> 接收到这个请求后,发现请求的这个资源实际存放在另一个位置
> 于是server在返回的response header的Location字段中写入那个请求资源的正确的URL，并设置reponse的状态码为30x
> client:
> 接收到这个response后,发现状态码为重定向的状态吗,就会去解析到新的URL,根据新的URL重新发起请求

## 状态码
 > 重定向最常用为301,也有303,
 > 临时重定向用302,307

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1781505-e7074166a674d155.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 与请求转发的对比
> 请求转发
>  服务器在处理request的过程中将request先后委托多个servlet或jsp接替进行处理的过程,request和reponse始终在期间传递

* 重定向时,客户端发起两次请求,而请求转发时,客户端只发起一次请求
* 重定向后,浏览器地址栏url变成第二个url,而请求转发没有变(请求转发对于客户端是透明的)
* 流程
> 重定向:
> 用户请求-----》服务器入口-------》组件------>服务器出口-------》用户----(**重定向**)---》新的请求
> 请求转发
> 用户请求-----》服务器入口-------》组件1---(**转发**)----》组件2------->服务器出口-------》用户

# 服务端实现
* 重定向
```java
response.sendRedirect（）
```
* 请求转发
  在Servlet里，实现dispatch是通过RequestDispatchers来实现的,这个类有两个方法:forward,include
  JSP里实现dispatch的标签也有两个：jsp:forward和jsp:include

# okhttp中的实现
## 逻辑
* 读取到响应状态码为30x时,拿到重定向的url,用这个url置换掉request中的原url
* 对request的一些header,body分情况进行处理
* 重新执行这个request,发出请求

## 代码实现
在[RetryAndFollowUpInterceptor](https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/http/RetryAndFollowUpInterceptor.java)的intercept方法中构建了while (true)循环,循环内部:
###1. 根据response重建request:

```
Request followUp = followUpRequest(response)
int responseCode = userResponse.code();
```

然后switch (responseCode)进行判断:

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1781505-2be116337e25b8e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1781505-04d9aa6c4f55c6c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.重新执行这个request:
将前一步得到的followUp 赋值给request,重新进入循环,

    @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    ...
    
    while (true) {
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }
    
      Response response = null;
      boolean releaseConnection = true;
      try {
        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);//执行网络请求
        releaseConnection = false;
     ...
    
      Request followUp = followUpRequest(response);//拿到重定向的request
       if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;//如果不是重定向,就返回response
      }
    
      ...
    if (++followUpCount > MAX_FOLLOW_UPS) {//有最大次数限制20次
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }
    
      request = followUp;//把重定向的请求赋值给request,以便再次进入循环执行
      priorResponse = response;
    }
    }


## okhttp的相关配置

> 重定向功能默认是开启的,可以选择关闭,然后去实现自己的重定向功能:

    		new OkHttpClient().newBuilder()  
                             .followRedirects(false)  //禁制OkHttp的重定向操作，我们自己处理重定向
                              .followSslRedirects(false)//https的重定向也自己处理

# 封装到极致的网络库:
[HttpUtilForAndroid](https://github.com/hss01248/HttpUtilForAndroid)