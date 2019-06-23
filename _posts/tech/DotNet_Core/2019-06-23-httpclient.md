---
layout: post
title: .Net Core下HttpClient的使用姿势
category: DotNet_Core
tags: httpclient
keywords: 
description: 
---

### 一 、回顾

在传统的FrameWork，.Neter 都习惯于通过如下方式来发送Http请求

```java 
using (var httpClient = new HttpClient())
{
    var response =
}

```

可是，当我们试图运行下面的测试：

```java
public async Task SendRequest() 
{
    Console.WriteLine("Starting reqeust");
    for(int i = 0; i<10; i++)
    {
        using(var client = new HttpClient())
        {
            var result = await client.GetAsync("http://www.baidu.com");
            Console.WriteLine(result.StatusCode);
        }
    }
    Console.WriteLine("Reqeust done");
}
```

此时在terminal下列出所有端口:

![image.png-165.2kB][1]


  [1]: http://static.zybuluo.com/qxjbeyond/3lx1on6bnvvkeb5hwtvegfic/image.png
  
  
  所以，如果你把HttpClient用作大规模的Http请求，实际上会创建很多个Http连接，而且这些资源并不能被立即释放。
  
  
### 二、解决问题
 
 1、单例，用单例的HttpClient的确会减少Socket资源。
 
 但是这个方案会引发新的问题：由于这个Http连接始终保持连接状态，所以 当请求地址的DNS发生更新的时候并不会应用到这个Http连接上。
 
 2、HttpClientFactory
 
  一个叫做HttpClientFactory的开源实现用来彻底解决这个问题。微软也将HttpClientFactory集成在了.NET Core中。使用方法如下：
  
  
 No1. 添加Nuget包
  
```java
   Microsoft.Extensions.Http
   
```
  
 No2. 注册服务
 
```java
 services.AddHttpClient();
 
```
 
 No3.简单示例
 
```java
public class BasicUsage
{
    private readonly IHttpClientFactory _clientFactory;
 
    public BasicUsage(IHttpClientFactory clientFactory)
    {
        _clientFactory = clientFactory;
    }
 
    public async Task SendRequest()
    {
        var request = new HttpRequestMessage(HttpMethod.Get, 
            "http://www.baidu.com");
 
        var client = _clientFactory.CreateClient();
        var response = await client.SendAsync(request);
        //do something for response
    }
}

```

3、 使用Named HttpClient

> 如果你需要不同配置的HttpClient，你可以通过“起名字的”的方式注册不同的HttpClient。


```java
services.AddHttpClient("test", c =>
{
    c.BaseAddress = new Uri("https://www.baidu.com");
    c.DefaultRequestHeaders.Add("Accept", "application/json");
});
```

一旦注册了一个名叫“test"的HttpClient，你就可以通过下面的方式来创建HttpClient：

```java
var client = _clientFactory.CreateClient("baidu");

```


### 三、 集成 polly

> Polly 是适用于 .NET 的全面恢复和临时故障处理库。 开发人员通过它可以表达策略，例如以流畅且线程安全的方式处理重试、断路器、超时、Bulkhead 隔离和回退。


安装polly Nuget包


```java
Microsoft.Extensions.Http.Polly
```


No1:处理临时故障

```java
services.AddHttpClient<UnreliableEndpointCallerService>()
    .AddTransientHttpErrorPolicy(p => 
        p.WaitAndRetryAsync(3, _ => TimeSpan.FromMilliseconds(600)));
```
上述代码中定义了 WaitAndRetryAsync 策略。 请求失败后最多可以重试三次，每次尝试间隔 600 ms。



No2:动态选择策略

```java
var timeout = Policy.TimeoutAsync<HttpResponseMessage>(
    TimeSpan.FromSeconds(10));
var longTimeout = Policy.TimeoutAsync<HttpResponseMessage>(
    TimeSpan.FromSeconds(30));

services.AddHttpClient("conditionalpolicy")
// Run some code to select a policy based on the request
    .AddPolicyHandler(request => 
        request.Method == HttpMethod.Get ? timeout : longTimeout);
```
在上述代码中，如果出站请求为 HTTP GET，则应用 10 秒超时。 其他所有 HTTP 方法应用 30 秒超时。


No3:添加多个 Polly 处理程序

```java
services.AddHttpClient("multiplepolicies")
    .AddTransientHttpErrorPolicy(p => p.RetryAsync(3))
    .AddTransientHttpErrorPolicy(
        p => p.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
```
 第一个使用 AddTransientHttpErrorPolicy 扩展添加重试策略。 若请求失败，最多可重试三次。 第二个调用 AddTransientHttpErrorPolicy 添加断路器策略。 如果尝试连续失败了五次，则会阻止后续外部请求 30 秒。 断路器策略处于监控状态。 通过此客户端进行的所有调用都共享同样的线路状态。

No4:从 Polly 注册表添加策略


```java
var registry = services.AddPolicyRegistry();

registry.Add("regular", timeout);
registry.Add("long", longTimeout);

services.AddHttpClient("regulartimeouthandler")
    .AddPolicyHandlerFromRegistry("regular");
```
两个策略在 PolicyRegistry 添加到 ServiceCollection 中时进行注册。 若要使用注册表中的策略，请使用 AddPolicyHandlerFromRegistry 方法，同时传递要应用的策略的名称。



### 四、HttpClient的简单封装

```java
public  class HttpClientHelper
    {
       
        //写法1，使用单例模式
        public async Task<string> SendAsync(string url, HttpMethod method, string reqParams = "")
        {
            using (HttpRequestMessage reqMessage = new HttpRequestMessage(method, url))
            {
                reqMessage.Content = new StringContent(reqParams, Encoding.UTF8, "application/json");

                using (var resMessage = await HttpClientSigleton.Instance.httpClient.SendAsync(reqMessage))
                {
                    resMessage.EnsureSuccessStatusCode();
                    return await resMessage.Content.ReadAsStringAsync().ConfigureAwait(false);
                }
            }
        }


    public class HttpClientSigleton:Singleton<HttpClientSigleton>
        {
        public HttpClient httpClient = null;
        public HttpClientSigleton()
         {
            HttpClientHandler handler = new HttpClientHandler()
            {
                Proxy = null,
                UseCookies = false,
                AllowAutoRedirect = false,
                AutomaticDecompression = DecompressionMethods.GZip | DecompressionMethods.Deflate,
            };
            httpClient = new HttpClient(handler);
            httpClient.DefaultRequestHeaders.ExpectContinue = false;
            httpClient.DefaultRequestHeaders.Add("Accept", "application/json");
         }
       }

        //写法2  asp.Net Core 推荐的方式
        private readonly IHttpClientFactory _clientFactory;

        public HttpClientHelper(IHttpClientFactory clientFactory)
        {
            _clientFactory = clientFactory;
        }
        public async Task<string> SendAsyncCore(string url, HttpMethod method, string reqParams = "")
        {
            using (HttpRequestMessage reqMessage = new HttpRequestMessage(method, url))
            {
                reqMessage.Content = new StringContent(reqParams, Encoding.UTF8, "application/json");

                using (var resMessage = await _clientFactory.CreateClient("kmauth").SendAsync(reqMessage))
                {
                    resMessage.EnsureSuccessStatusCode();
                    return await resMessage.Content.ReadAsStringAsync().ConfigureAwait(false);
                }
            }
        }


    }
```
