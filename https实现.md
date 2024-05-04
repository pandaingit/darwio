# 服务之间https通信

## 前言

目前服务之前通信是通过获取nacos注册ip和端口之后,进行http通信, 存在被窃听, 更改信息的风险

本文档主要说明如何在服务之间进行https通信, 避免http协议的设计

 

## 方案设计

 

一.服务本身支持https

 

### tomcat配置ssl

tomcat, springgateway自带支持ssl, 进行nups协议通信

1. 通过java自中Keytool生成java keystrore(JKS)格式的证书文件

```
Keytool -genkey -keypass 123456 -keystore system.keystore

Keytool -importkeystore -srckeystore system.keystore -destkeystore system.keystore -deststoretype pkcs12
```



可以通过openssl将CRT转成JKS

```
#根据server,crt和server.key生成pkcs12i证书
openssl pkas12 -export -in server.crt-inkey server.ckey -out mycert. p12-nme abc -CAfile myCA.crt

#P12证书转换为jks证书
keytool -importheystore -v -srckeystore mycert. p12. -srcstoretype pkcS12 -srcstorepass 123456 -destkeystore server.key
```

 

2.配置服务启动https

```
server:
	ssl:
       enabled: true#开始ssl认证
       key-store: classpath :server.keystore F证书位苦key-store-type: Jks#秘钥库存储类型
key-store-type:JKS #秘钥库存储类型
       key-store-password: 123456#秘钥库口令
```



### nginx代理实现https

> 服务采用nginx反向代理的形式来实现https协议通信

生成ssl证书:

```
openssl genrsa -dess -out server key 2048
openssl rsa -in server.key-out serven.key
openssl reg -new -key server.key -out server.csr
openss1 reg -new -x509 -key server.key-out ca.crt -days 3650
openssl x509 -req -days 3650 -in server.csr -CA ca.crt -CAkey server.key-CAcreateserial -out server.crt
```



1.启动https

```
upstream server16001 {
	server 10.17.6.78:16001 weight=1:
}
server {
	listen 16001 ssl;
	server_name mytest.duoyi.com:
	ssl_certaficate /etc/nginx/ss1/server.crt
	ssl_certificate_key /etc/nginx/ss1/server.key;
	1ocation / {
		proxy_pass http://server16001;
	}
}
```

2. 本地配置host

```
mytest.duoyi.com 127.0.0.1
```





## 验证

### 调用服务采用https

 

有两种形式可以进行https调用

1. feign中进行value配置

```
@service
@Feignclient(
    value = "https://zhaopin-domain-support-permission-service",
)
```

 

2. nacos注册说明当前服务为https(推荐)

```
spring:
    cloud:
       nacos:
           discovery:
              secure: true
```

这样原有的feign进行调用时,最后解析就会默认加上https

 

### 信任证书

> 由于本地生成的https证书无法在调用时进行认证， 因而需要手动配置信任证书进行测试



1.配置信任证书

```
#生成truststore
keytool -import -file zp40.keystore -keystore serverTrust.keystore
```

2.配置feignclient

```
server:
	ssl:
		trust-store: classpath:serverTrust.keystore#证书位置
		trust-store-password: 123456
		trust-store-type: JKS #信任库秘钥存储类型
```

调整注册的feignclient  sslcontext

```
@Value("${server.ssl.trust-store}")
private string trustStore;
@Value("${server.ss1 .trust-store-password}")
private String trustStorePass;

/**
 * 配置ssl信任证书
 */
@Bean
public SSLContext sslcontext() throws Exception {
	return ssLContextBuilder.create()
		.loadTrustMaterial(Resourceutils.getFile(trustStore), trustStorePass.tocharArray())build()
}
```

