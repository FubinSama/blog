# Spring Cloud Gateway无法支持gzip问题分析

## 背景

我司通过对Spring Cloud Gateway二次开发，实现一套可以进行加解密和混淆的网关，命名为Decrypt Gateway，它会对请求进行处理，转发到对应的业务服务，如：Loan Center。在预发和生产环境上我们会通过运维在nginx配置的内部域名（正向代理）来配置这些业务服务的地址。

但是运维nginx默认为开启gzip压缩，之前

## 探索

### 探索前准备

本机启动nginx，并配置正向代理指向测试服务器上的loanCenter服务

## 实验

### 实验前准备

网关增加配置httpClient开启compress

```java
@Configuration
public class HttpClientConfigure implements HttpClientCustomizer {
    
    @Override
    public HttpClient customize(HttpClient httpClient) {
        return httpClient
                .compress(true); // enable the compression for HTTP
    }
    
}
```

### nginx反向代理关闭gzip



### nginx反向代理开启gzip

###  