---
title: 快速启动
date: '2021-09-24'
type: book
weight: 10
---

快速安装并运行Biz-SIP中间件，并运行相关的测试案例.

<!--more-->

## 一、安装
### 1 项目源码下载

1. 从“[https://gitee.com/szhengye/biz-sip](https://gitee.com/szhengye/biz-sip)”中Clone下载项目源码，SpringBoot版本下载springboot分支，SpringCloud版本下载springcloud分支。
1. 在Eclipse或IDEA中作为Maven项目导入。
### 2 MySQL安装

1. 安装MySQL镜像
```shell
docker run -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=bizsip123456 -d mysql
```

2. 建库

执行项目中sql/sample.sql脚本，以建立sip演示库。
### 3 Redis安装
```shell
docker run -p 6379:6379 -d redis:latest redis-server
```
### 4 RabbitMQ安装
```shell
docker pull rabbitmq
docker run -d --name rabbitmq -e RABBITMQ_DEFAULT_USER=springcloud -e RABBITMQ_DEFAULT_PASS=springcloud -p 15672:15672 -p 5672:5672 rabbitmq:management
# 下载延迟队列插件，并拷贝到容器中相关目录
docker cp rabbitmq_delayed_message_exchange-3.8.0.ez rabbitmq:/plugins
# 进入容器交互终端模式，用"docker ps"命令查询RabbitMQ的容器ID
docker exec -it <容器ID> bash
# 执行命令启用延迟队列插件
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```
### 5 Nacos安装（SpringCloud版本）

1. 下载Nacos镜像
```shell
docker pull nacos/nacos-server
```

2. 启动Nacos镜像
```shell
docker run --env MODE=standalone --name nacos -d -p 8848:8848 nacos/nacos-server
# 登录密码默认nacos/nacos
# standalone 代表单机模式运行，非集群模式
```

3. 访问Nacos，默认账号/密码：nacos/nacos，访问地址：[http://localhost:8848/nacos/index.html](http://192.168.247.131:8848/nacos/index.html)
## 二、配置
### 1 integrator相关配置文件
application.yml文件：
```yaml
spring:
  profiles:
    active: local
```
application-local.yml文件：
```yaml
server:
  port: 8888

spring:
  application:
    name: bizsip-integrator

  cloud:
    nacos:
      discovery:
        server-addr: bizsip-nacos:8848
#  以下配置在Istio部署中打开，以不采用NACOS注册中心，而采用etcd注册机制
#  cloud:
#    service-registry:
#      auto-registration:
#        enabled: false  #禁用注册中心服务
        
  datasource:
    url: jdbc:mysql://bizsip-mysql/sip
    username: root
    password: bizsip123456
    driver-class-name: com.mysql.jdbc.Driver

  redis:
    redisson:
      enable: true
    host: bizsip-redis
    port: 6379
    timeout: 6000
    database: 0
    lettuce:
      pool:
        max-active: 10 # 连接池最大连接数（使用负值表示没有限制）,如果赋值为-1，则表示不限制；如果pool已经分配了maxActive个jedis实例，则此时pool的状态为exhausted(耗尽)
        max-idle: 8   # 连接池中的最大空闲连接 ，默认值也是8
        max-wait: 100 # # 等待可用连接的最大时间，单位毫秒，默认值为-1，表示永不超时。如果超过等待时间，则直接抛出JedisConnectionException
        min-idle: 2    # 连接池中的最小空闲连接 ，默认值也是0
      shutdown-timeout: 100ms

  rabbitmq:
    virtual-host: /
    host: bizsip-rabbitmq
    port: 5672
    username: springcloud
    password: springcloud
    listener:
      simple:
        concurrency: 5
        max-concurrency: 15
        prefetch: 10

bizsip:
  config-path: /var/bizsip/config

logging:
  level:
    com.bizmda.bizsip: info
```
 其中：

- server.port：服务整合器（Integrator）的微服务端口，建议用8888端口，避免和其它端口相冲突。
- spring.datasource.*：为数据库相关配置，应连接对应的数据库。
- spring.redis.*：为Redis相关配置，连接对应的Redis。
- spring.rabbitmq.*：为RabbitMQ相关配置，连接RabbitMQ中间件。
- bizsip.config-path：配置成项目下sample/config的实际安装目录。
- logging.level.com.bizmda.bizsip：日志级别，一般设置为info。
### 2 sample/sample-sink模块相关配置文件
application.yml文件：
```yaml
spring:
  profiles:
    active: local
```
application-local.yml文件：
```yaml
server:
  port: 8001
spring:
  application:
    name: bizsip-sample-sink

  cloud:
    nacos:
      discovery:
        server-addr: bizsip-nacos:8848
  #  以下配置在Istio部署中打开，以不采用NACOS注册中心，而采用etcd注册机制
  #  cloud:
  #    service-registry:
  #      auto-registration:
  #        enabled: false  #禁用注册中心服务

  rabbitmq:
    virtual-host: /
    host: bizsip-rabbitmq
    port: 5672
    username: springcloud
    password: springcloud
    listener:
      simple:
        concurrency: 5
        max-concurrency: 15
        prefetch: 10

bizsip:
  config-path: /Users/shizhengye/IdeaProjects/biz-sip/sample/config
#  config-path: /var/bizsip/config
#  sink-id: sink-nothing	#如果引用了com.bizmda.bizsip.sink.controller.SinkController，就需要配置这个参数

logging:
  level:
    com.bizmda.bizsip: debug
```
其中：

- server.port：为sample服务接入模块的RESTful服务端口，建议用8080。
- spring.rabbitmq.*：为RabbitMQ相关配置，连接RabbitMQ中间件。
- bizsip.config-path：配置成项目下sample/config的实际安装目录。
- sink-id：如果引用了Sink默认Controller（com.bizmda.bizsip.sink.controller.SinkController），就需要配置这个参数
- logging.level.com.bizmda.bizsip：日志级别，一般设置为info。
### 3 sample/sample-source模块相关配置文件
application.yml文件：
```yaml
spring:
  profiles:
    active: local
```
application-local.yml文件：
```yaml
server:
  port: 8080

spring:
  application:
    name: bizsip-sample-source

  cloud:
    nacos:
      discovery:
        server-addr: bizsip-nacos:8848
  #  以下配置在Istio部署中打开，以不采用NACOS注册中心，而采用etcd注册机制
  #  cloud:
  #    service-registry:
  #      auto-registration:
  #        enabled: false  #禁用注册中心服务
  
bizsip:
  config-path: /var/bizsip/config
  integrator-url: http://bizsip-integrator/api

logging:
  level:
    com.bizmda.bizsip: info
```
其中：

- server.port：为sample服务调用模块的微服务端口，建议用8001，不要其它端口相冲突。
- bizsip.integrator-url：为服务整合器（Integrator）的服务接口url，原则上不需要修改。
- bizsip.config-path：配置成项目下sample/config的实际安装目录。
- logging.level.com.bizmda.bizsip：日志级别，一般设置为info。
### 4 source/netty-source模块相关配置文件
application.yml文件：
```yaml
spring:
  profiles:
    active: local
```
application-local.yml文件：
```yaml
server:
  port: 8090

spring:
  application:
    name: bizsip-netty-source

  cloud:
    nacos:
      discovery:
        server-addr: bizsip-nacos:8848
  #  以下配置在Istio部署中打开，以不采用NACOS注册中心，而采用etcd注册机制
  #  cloud:
  #    service-registry:
  #      auto-registration:
  #        enabled: false  #禁用注册中心服务

bizsip:
  config-path: /var/bizsip/config
  integrator-url: http://bizsip-integrator/api
  source-id: source1

netty:
  port: 10002

logging:
  level:
    com.bizmda.bizsip: debug
```
其中：

- server.port：为sample服务调用模块的微服务端口，建议用8090，不要其它端口相冲突。
- bizsip.integrator-url：为服务整合器（Integrator）的服务接口url，原则上不需要修改。
- bizsip.config-path：配置成项目下sample/config的实际安装目录。
- bizsip.source-id：为当前source模块的Source ID。
- logging.level.com.bizmda.bizsip：日志级别，一般设置为info。
## 三、启动系统
测试用例架构如下图所示：

![image.png](1619764437376-05aaaef1-f349-4193-8b4a-2bff736c10b3-20210924201314876.png)

### 1 启动服务整合器（Integrator）
运行integrator模块下“IntegratorApplication.java”
### 2 启动Sample Sink模块
运行sample/sample-sink模块下“SampleSinkApplication.java”
### 3 启动Sample Source模块
运行sample/sample-source模块下“SampleSourceApplication.java”
### 4 启动Netty TCP Source模块（可选）
运行source/netty-source模块下“NettySourceApplication.java”
### 5 启动OpenAPI网关（SpringCloud版本，可以不用开启）
运行api-gateway模块下“GatewayApplication.java”。
对于后续测试案例的调用发起，都是直接发给Integrator服务整合器的，端口是8888；如果需要从OpenAPI网关发起，端口是8000。这二种情况的返回结果是一致的。
后续测试案例都是直接发给Integrator服务整合器的，目前端口是8888。
## 四、运行Sample测试案例
### 1 简单服务处理
通过Biz-SIP的开放API接口发送请求，由服务整合器处理后，直接返回。
![](a51ec1a83e2a9a313c8f9c31ed2663de.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/sample1" -X POST --data '{"accountNo":"62001818","sex":"0","email":"123232@163.com","mobile":"18601872345"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "414ac54e35d446a9bf36a7bae189eb2b",
  "parentTraceId": null,
  "timestamp": 1610334191563,
  "data": {
    "accountName": "62001818的账户名...",
    "sex": "0",
    "mobile": "18601872345",
    "accountNo": "62001818",
    "email": "123232@163.com"
  }
}
```
### 2 简单服务转发
通过Biz-SIP的开放API接口，调用CRM服务（用CrmServer.java类作为模拟），查询返回对应的账户名称。
![](ea627b4bd4f84a360a6ab7657e531530.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/sample2" -X POST --data '{"accountNo":"003"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "f30a96f6d73b4f44853b56a29688e1aa",
  "parentTraceId": null,
  "timestamp": 1609914563646,
  "data": {
    "accountName": "张三",
    "accountNo": "003"
  }
}
```
### 3 非标接口接入
通过非标API接口上送服务请求，实现原来通过标准API接口调用openapi/sample2服务。
![](6a00eb4215d03a586d2e7effc3ef4536.svg)

```shell
$ curl -H "Content-Type:application/json" -X POST --data '{"serviceId":"/openapi/sample2","accountNo":"003"}' http://localhost:8080/source1|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "9a0421faccf14f7b8f8dfd9916e8aca2",
  "parentTraceId": null,
  "timestamp": 1619768390392,
  "data": "{\"accountName\":\"张三\",\"accountNo\":\"003\",\"serviceId\":\"/openapi/sample2\"}"
}

```
### 4 调用多个服务
通过Biz-SIP的开放API接口，调用Server1返回帐户名称，调用Server2返回账户余额。
![](224ccd5fb77e8fc5651151dc0d238340.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/sample4" -X POST --data '{"accountNo":"005"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "67e451ea394a4714b3be6b4496ead1e6",
  "parentTraceId": null,
  "timestamp": 1609914708687,
  "data": {
    "accountName": "王五",
    "balance": 500,
    "accountNo": "005"
  }
}
```
### 5 简单XML消息转换
通过Biz-SIP的开放API接口，用simple-xml消息格式调用Server3，Server3会调用EchoServer打印消息报文，并用原服文返回。
![](0d42cc640f7ae8bda9d635aa9b5eda28.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/sample5" -X POST --data '{"accountName": "王五","balance": 500,"accountNo":"005"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "41112613fd4d4d9a85d8ba420ed5dbd5",
  "parentTraceId": null,
  "timestamp": 1609914764308,
  "data": {
    "accountName": "王五",
    "balance": 500,
    "accountNo": "005"
  }
}
```
Server3接入EchoServer类打印的消息：
```xml
EchoServer传入消息:<?xml version="1.0" encoding="UTF-8" standalone="no"?><root><accountName>王五</accountName><balance>500</balance><accountNo>005</accountNo></root>
```
### 6 复杂JSON消息转换
通过Biz-SIP的开放API接口，用velocity-json消息格式调用Server4，Server4会调用EchoServer打印消息报文，并用原服文返回。
![](bdc48a9317d500998810b7c44929cbfa.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/server4" -X POST --data '{"accountName": "王五","sex": "0","accountNo":"005"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "f6e36e068f824d57840222bafb9f0fac",
  "parentTraceId": null,
  "timestamp": 1609914809605,
  "data": {
    "sex": "女",
    "account_no": "005",
    "account_name": "王五"
  }
}
```
EchoServer打印日志：
```shell
EchoServer传入消息:{sex:"女",account_no:"005",account_name:"王五"}
```


```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/server4" -X POST --data '{"accountName": "王五","sex": "1","accountNo":"005"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "f2d0e28597634b1a89c4e8e4140ae8a7",
  "parentTraceId": null,
  "timestamp": 1609914847120,
  "data": {
    "sex": "男",
    "account_no": "005",
    "account_name": "王五"
  }
}
```
EchoServer打印日志：
```shell
EchoServer传入消息:{sex:"男",account_no:"005",account_name:"王五"}
```


```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/server4" -X POST --data '{"accountName": "王五","sex": "2","accountNo":"005"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "297c62d7fc234eb894d7447e0667f360",
  "parentTraceId": null,
  "timestamp": 1609914874611,
  "data": {
    "sex": "?",
    "account_no": "005",
    "account_name": "王五"
  }
}
```
EchoServer打印日志：
```shell
EchoServer传入消息:{sex:"?",account_no:"005",account_name:"王五"}
```
### 7 复杂XML消息转换
通过Biz-SIP的开放API接口，用velocity-xml消息格式调用Server5，Server5会调用EchoServer打印消息报文，并用原服文返回。
![](18e5d62adf6d0e070012e927de0aabee.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/server5" -X POST --data '{"accountName": "王五","sex": "0","accountNo":"005","balance":100}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "04a92806779a415b94a056a390c38c58",
  "parentTraceId": null,
  "timestamp": 1631418837236,
  "data": {
    "account": {
      "no": "005",
      "sex": "女人",
      "balance": 100,
      "name": "王五"
    }
  }
}
```


SampleSinkApplication打印日志：
```xml
EchoServer传入消息:[<account>
    <name>王五</name>
    <balance>100</balance>
    <no>005</no>
    <sex>女人</sex>
</account>]
```
```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/server5" -X POST --data '{"accountName": "王五","sex": "0","accountNo":"005","balance":1000}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "c798f6cfc22b42e19933d860ba6bce3b",
  "parentTraceId": null,
  "timestamp": 1609915845068,
  "data": {
    "account": {
      "no": "005",
      "sex": "女人",
      "balance": 1000,
      "name": "王五"
    }
  }
}
```
SampleSinkApplication打印日志：
```shell
EchoServer传入消息:[<account>
    <name>王五</name>
    <balance>1000</balance>
    <no>005</no>
    <sex>女人</sex>
</account>]
```


```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/server5" -X POST --data '{"accountName": "王五","sex": "2","accountNo":"005","balance":1000}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "8014463d1c7d451cb47c279973931ba0",
  "parentTraceId": null,
  "timestamp": 1609915967831,
  "data": {
    "account": {
      "no": "005",
      "sex": "不知道",
      "balance": 1000,
      "name": "王五"
    }
  }
}
```
SampleSinkApplication打印日志：
```xml
EchoServer传入消息:[<account>
    <name>王五</name>
    <balance>1000</balance>
    <no>005</no>
    <sex>不知道</sex>
</account>
```
### 8 定长消息转换
通过Biz-SIP的开放API接口，用定长消息格式调用Server6，Server6会调用EchoServer打印消息报文，并用原服文返回。
![](99b3cddca4737803fc0453b97bea2e9b.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/server6" -X POST --data '{"accountName": "王五","balance": 500,"accountNo":"005","sex":"0"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "f46a1f79ca0241719b4dc1bd3cc3457c",
  "parentTraceId": null,
  "timestamp": 1609916044599,
  "data": {
    "accountName": "王五        ",
    "sex": "0",
    "balance": "500       ",
    "accountNo": "005     "
  }
}

```
SampleSinkApplicationc对于Server6接入EchoServer类打印的消息：
```shell
EchoServer传入消息:[0005     王五        500       ]
```
### 9 有分隔符消息转换
通过Biz-SIP的开放API接口，用定长消息格式调用Server7，Server7会调用EchoServer打印消息报文，并用原服文返回。
![](11d78198c66f5e6b2f3ef974a78cd311.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/server7" -X POST --data '{"accountName": "王五","balance": 500,"accountNo":"005","sex":"0"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "3a139d684acc4137b744625cace14267",
  "parentTraceId": null,
  "timestamp": 1610336392183,
  "data": {
    "array": [
      [
        "0",
        "005"
      ],
      [
        "王五",
        "500"
      ]
    ]
  }
}
```
SampleSinkApplicationc对于Server7接入EchoServer类打印的消息：
```shell
EchoServer传入消息:[0,005*王五,500]
```
### 10 ISO8583消息转换
通过source controller，在sink端打包成ISO8583报文格式调用sink13，sink13会调用EchoServer打印消息报文，并用原服文返回，在sink端解包成JSON报文后返回。
![](4cb75220113a8cd22309c73fd6d4b483.svg)

```shell
$ curl -H "Content-Type:application/json" http://localhost:8080/sink13|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "91bef734f69a4da8b1be35138a4e9707",
  "parentTraceId": null,
  "timestamp": 1632041946033,
  "data": {
    "msgType": "0800",
    "reserved60": "000000000030",
    "card_accptr_id42": "898411341310014",
    "systemTraceAuditNumber11": "000001",
    "switching_data62": "53657175656e6365204e6f3132333036303733373832323134",
    "card_accptr_termnl_id41": "73782214",
    "msgHead": "31323031383430303031303334342020203432343330343430202020393031323334353637383930313233343536",
    "acct_id_1_102": "1234567890",
    "fin_net_data63": "303031"
  }
}
```
SampleSinkApplicationc对于sink13接入EchoServer类打印的消息：
### 11 调用Netty服务端
通过Biz-SIP的开放API接口，调用Netty TCP服务端。
![](1cf2a7af1d0d0ff59b00eddf31d350f4.svg)

在终端1启动：

```shell
$ echo '{"accountName": "王五","balance": 500,"accountNo":"xxx"}'|nc -l 10001
```
在终端2启动：
```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/netty" -X POST --data '{"accountNo":"999"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "bb4255801a54442ca16f0189d64b951f",
  "parentTraceId": null,
  "timestamp": 1609916153979,
  "data": {
    "accountName": "王五",
    "balance": 500,
    "accountNo": "xxx"
  }
}
```
终端1会显示：
```shell
$ echo '{"accountName": "王五","balance": 500,"accountNo":"xxx"}'|nc -l 10001
{"accountNo":"999"}
```
### 12 Netty客户端接入
通过Netty客户端接入，来调用聚合服务。
![](67782a1d38f339e0eb42acc0d9e4f2ec.svg)

```shell
$ echo '{"serviceId":"/openapi/sample2","accountNo":"003"}'|nc localhost 10002

{"accountName":"张三","accountNo":"003","serviceId":"/openapi/sample2"}
```
### 13 调用数据库操作
通过Biz-SIP的开放API接口，在脚本服务中根据上送的账号在数据表中进行查询，并直接返回查询结果。
![](f06d1db2f470eb1fda53e4e408adbc25.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/db1" -X POST --data '{"accountNo":"002"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "19dd46603d844b40a1bce11cd2760e1c",
  "parentTraceId": null,
  "timestamp": 1609924306042,
  "data": [
    {
      "accountName": "李四",
      "sex": "1",
      "balance": 200,
      "accountNo": "002"
    }
  ]
}

```
### 14 调用Redis操作
通过Biz-SIP的开放API接口，在脚本服务中根据上送的账号在Redis中进行存储，并直接返回查询结果。
![](a3fabc46733738ac91614188eec32763.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/redis1" -X POST --data '{"accountNo":"002"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "e04430bb7f754959b52a28a8a6e4456a",
  "parentTraceId": null,
  "timestamp": 1609924818739,
  "data": {
    "accountName": "002",
    "accountNo": "002"
  }
}
```
```shell
redis>get account_no
"002"
```
### 15 通过事务管理进行服务调用重试
通过Biz-SIP的开放API接口，在脚本服务中调用SAF服务，对server3服务进行多达5次的服务重试调用。
![](2d025639c899667263b26120319031d1.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/safservice" -X POST --data '{"accountNo":"003"}' http://localhost:8888/api|jq  
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   186    0   167  100    19    598     68 --:--:-- --:--:-- --:--:--   666
{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "f1afe72dac0842ad8912e063b7bcf593",
  "parentTraceId": null,
  "timestamp": 1631513601741,
  "data": {
    "accountNo": "003"
  }
}
```

SampleSinkApplication日志：
```
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:21.952 DEBUG 53605 [48E46F14E87845BE9040414A89AEFC03] [http-nio-8001-exec-1] c.b.b.s.sink.controller.Sink3Controller  inMessage:BizMessage(code=0, message=null, extMessage=null, traceId=f1afe72dac0842ad8912e063b7bcf593, parentTraceId=null, timestamp=1631513601741, data={"retryCount":1,"accountNo":"003"})
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:21.955 DEBUG 53605 [48E46F14E87845BE9040414A89AEFC03] [http-nio-8001-exec-1] com.bizmda.bizsip.sink.Sink              Sink调用Convert[simple-xml]打包
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:21.963 DEBUG 53605 [48E46F14E87845BE9040414A89AEFC03] [http-nio-8001-exec-1] com.bizmda.bizsip.sink.Sink              Sink通过Connect[sink-bean]调用服务
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:21.963 DEBUG 53605 [48E46F14E87845BE9040414A89AEFC03] [http-nio-8001-exec-1] c.b.b.s.connector.SinkBeanSinkConnector  调用SinkBeanSinkConnector[com.bizmda.bizsip.sample.sink.controller.EchoServer]的process()
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:21.963 DEBUG 53605 [48E46F14E87845BE9040414A89AEFC03] [http-nio-8001-exec-1] c.b.b.sample.sink.controller.EchoServer  EchoServer传入消息:[<?xml version="1.0" encoding="UTF-8" standalone="no"?><root><retryCount>1</retryCount><accountNo>003</accountNo></root>]
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:21.964 DEBUG 53605 [48E46F14E87845BE9040414A89AEFC03] [http-nio-8001-exec-1] com.bizmda.bizsip.sink.Sink              Sink调用Convert[simple-xml]解包
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:24.014 DEBUG 53605 [317BD9F4F3044E92A6A1D271ACD8376B] [http-nio-8001-exec-2] c.b.b.s.sink.controller.Sink3Controller  inMessage:BizMessage(code=0, message=null, extMessage=null, traceId=f1afe72dac0842ad8912e063b7bcf593, parentTraceId=null, timestamp=1631513601741, data={"retryCount":2,"accountNo":"003"})
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:24.014 DEBUG 53605 [317BD9F4F3044E92A6A1D271ACD8376B] [http-nio-8001-exec-2] com.bizmda.bizsip.sink.Sink              Sink调用Convert[simple-xml]打包
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:24.028 DEBUG 53605 [317BD9F4F3044E92A6A1D271ACD8376B] [http-nio-8001-exec-2] com.bizmda.bizsip.sink.Sink              Sink通过Connect[sink-bean]调用服务
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:24.028 DEBUG 53605 [317BD9F4F3044E92A6A1D271ACD8376B] [http-nio-8001-exec-2] c.b.b.s.connector.SinkBeanSinkConnector  调用SinkBeanSinkConnector[com.bizmda.bizsip.sample.sink.controller.EchoServer]的process()
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:24.029 DEBUG 53605 [317BD9F4F3044E92A6A1D271ACD8376B] [http-nio-8001-exec-2] c.b.b.sample.sink.controller.EchoServer  EchoServer传入消息:[<?xml version="1.0" encoding="UTF-8" standalone="no"?><root><retryCount>2</retryCount><accountNo>003</accountNo></root>]
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:24.030 DEBUG 53605 [317BD9F4F3044E92A6A1D271ACD8376B] [http-nio-8001-exec-2] com.bizmda.bizsip.sink.Sink              Sink调用Convert[simple-xml]解包
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:28.083 DEBUG 53605 [A62381D250A149E8AC201E5F1BBBDE33] [http-nio-8001-exec-3] c.b.b.s.sink.controller.Sink3Controller  inMessage:BizMessage(code=0, message=null, extMessage=null, traceId=f1afe72dac0842ad8912e063b7bcf593, parentTraceId=null, timestamp=1631513601741, data={"retryCount":3,"accountNo":"003"})
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:28.083 DEBUG 53605 [A62381D250A149E8AC201E5F1BBBDE33] [http-nio-8001-exec-3] com.bizmda.bizsip.sink.Sink              Sink调用Convert[simple-xml]打包
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:28.091 DEBUG 53605 [A62381D250A149E8AC201E5F1BBBDE33] [http-nio-8001-exec-3] com.bizmda.bizsip.sink.Sink              Sink通过Connect[sink-bean]调用服务
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:28.091 DEBUG 53605 [A62381D250A149E8AC201E5F1BBBDE33] [http-nio-8001-exec-3] c.b.b.s.connector.SinkBeanSinkConnector  调用SinkBeanSinkConnector[com.bizmda.bizsip.sample.sink.controller.EchoServer]的process()
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:28.091 DEBUG 53605 [A62381D250A149E8AC201E5F1BBBDE33] [http-nio-8001-exec-3] c.b.b.sample.sink.controller.EchoServer  EchoServer传入消息:[<?xml version="1.0" encoding="UTF-8" standalone="no"?><root><retryCount>3</retryCount><accountNo>003</accountNo></root>]
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:28.093 DEBUG 53605 [A62381D250A149E8AC201E5F1BBBDE33] [http-nio-8001-exec-3] com.bizmda.bizsip.sink.Sink              Sink调用Convert[simple-xml]解包
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:36.162 DEBUG 53605 [305FB08B640B41EC92EF8AED3AE667F6] [http-nio-8001-exec-4] c.b.b.s.sink.controller.Sink3Controller  inMessage:BizMessage(code=0, message=null, extMessage=null, traceId=f1afe72dac0842ad8912e063b7bcf593, parentTraceId=null, timestamp=1631513601741, data={"retryCount":4,"accountNo":"003"})
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:36.162 DEBUG 53605 [305FB08B640B41EC92EF8AED3AE667F6] [http-nio-8001-exec-4] com.bizmda.bizsip.sink.Sink              Sink调用Convert[simple-xml]打包
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:36.167 DEBUG 53605 [305FB08B640B41EC92EF8AED3AE667F6] [http-nio-8001-exec-4] com.bizmda.bizsip.sink.Sink              Sink通过Connect[sink-bean]调用服务
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:36.167 DEBUG 53605 [305FB08B640B41EC92EF8AED3AE667F6] [http-nio-8001-exec-4] c.b.b.s.connector.SinkBeanSinkConnector  调用SinkBeanSinkConnector[com.bizmda.bizsip.sample.sink.controller.EchoServer]的process()
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:36.167 DEBUG 53605 [305FB08B640B41EC92EF8AED3AE667F6] [http-nio-8001-exec-4] c.b.b.sample.sink.controller.EchoServer  EchoServer传入消息:[<?xml version="1.0" encoding="UTF-8" standalone="no"?><root><retryCount>4</retryCount><accountNo>003</accountNo></root>]
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:36.168 DEBUG 53605 [305FB08B640B41EC92EF8AED3AE667F6] [http-nio-8001-exec-4] com.bizmda.bizsip.sink.Sink              Sink调用Convert[simple-xml]解包
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:52.373 DEBUG 53605 [933B128B00DB4B4AB79EE69CD2C80009] [http-nio-8001-exec-5] c.b.b.s.sink.controller.Sink3Controller  inMessage:BizMessage(code=0, message=null, extMessage=null, traceId=f1afe72dac0842ad8912e063b7bcf593, parentTraceId=null, timestamp=1631513601741, data={"retryCount":5,"accountNo":"003"})
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:52.375 DEBUG 53605 [933B128B00DB4B4AB79EE69CD2C80009] [http-nio-8001-exec-5] com.bizmda.bizsip.sink.Sink              Sink调用Convert[simple-xml]打包
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:52.379 DEBUG 53605 [933B128B00DB4B4AB79EE69CD2C80009] [http-nio-8001-exec-5] com.bizmda.bizsip.sink.Sink              Sink通过Connect[sink-bean]调用服务
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:52.379 DEBUG 53605 [933B128B00DB4B4AB79EE69CD2C80009] [http-nio-8001-exec-5] c.b.b.s.connector.SinkBeanSinkConnector  调用SinkBeanSinkConnector[com.bizmda.bizsip.sample.sink.controller.EchoServer]的process()
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:52.379 DEBUG 53605 [933B128B00DB4B4AB79EE69CD2C80009] [http-nio-8001-exec-5] c.b.b.sample.sink.controller.EchoServer  EchoServer传入消息:[<?xml version="1.0" encoding="UTF-8" standalone="no"?><root><retryCount>5</retryCount><accountNo>003</accountNo></root>]
[bizsip-sample-sink:192.168.10.15:8001] 2021-09-13 14:13:52.380 DEBUG 53605 [933B128B00DB4B4AB79EE69CD2C80009] [http-nio-8001-exec-5] com.bizmda.bizsip.sink.Sink              Sink调用Convert[simple-xml]解包

```
### 16 Java聚合服务实现简单服务转发
通过Biz-SIP的开放API接口，通过Java开发的聚合服务，来调用CRM服务（用CrmServer.java类作为模拟），查询返回对应的账户名称。
![](19b889d9ff36ddde7abd839ca5bb2825.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:sample1.service1" -X POST --data '{"accountNo":"003"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "31cd857883974c8bae79ecf8ac12fdf0",
  "parentTraceId": null,
  "timestamp": 1610246650385,
  "data": {
    "accountName": "张三",
    "accountNo": "003"
  }
}
```
### 17 Java聚合服务调用基于Script的延迟服务
通过Biz-SIP的开放API接口，通过Java开发的聚合服务，调用SAF服务，对server3服务进行多达5次的服务重试调用。
![](2d025639c899667263b26120319031d1-20210924202701753.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/sample/service2" -X POST --data '{"accountNo":"003"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": null,
  "extMessage": null,
  "traceId": "7df1a4f93098402b902ba8eafec7bfcb",
  "parentTraceId": "c4a603917bb04bbc97d8041015dbe36c",
  "timestamp": 1610247196000,
  "data": {
    "accountNo": "003"
  }
}
```
### 18 调用Netty长连接异步返回
通过Biz-SIP的开放API接口，通过RabbitMQ连接器，调用Netty TCP长连接，并把原包通过TCP长连接异步返回。
![](341e167db6560e43b060d0ab2418fda7.svg)在终端1启动：

```shell
$ nc -l 4444|nc localhost 4445
```
在终端2启动：
```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/server9" -X POST --data '{"accountNo":"62001818","sex":"0","email":"123232@163.com","mobile":"18601872345"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "e40c37b3196044b5a9f5b5e9c699262d",
  "parentTraceId": null,
  "timestamp": 1621303728434,
  "data": {
    "sex": "0",
    "mobile": "18601872345",
    "accountNo": "62001818",
    "email": "123232@163.com"
  }
}
```
### 19 调用SpringBean服务
通过Biz-SIP的开放API接口，调用Spring容器Bean服务（用EchoBeanServer.java类作为模拟）。
![](65f37a65daf1bcf8bb0336876aee88e4.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/openapi/server10" -X POST --data '{"accountNo":"003"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "f30a96f6d73b4f44853b56a29688e1aa",
  "parentTraceId": null,
  "timestamp": 1609914563646,
  "data": {
    "accountNo": "003"
  }
}
```
### 20 聚合服务调用SinkClient接口服务
通过Biz-SIP的开放API接口，调用以Java接口方式暴露的SinkClient服务。
![](bbdd4654c04896b1fc7bd728e24e0477.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/sample/service3" -X POST --data '{"accountNo":"003"}' http://localhost:8888/api|jq 

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "8490e767f25d4a08ad059a5ed49cec1d",
  "parentTraceId": null,
  "timestamp": 1631442018752,
  "data": {
    "customerDTO": {
      "sex": "1",
      "isMarried": false,
      "customerId": "001",
      "name": "张三",
      "age": 20
    },
    "accountDTOs": [
      {
        "balance": 1200,
        "account": "0001"
      },
      {
        "balance": 45000,
        "account": "0003"
      }
    ],
    "sink1": {
      "accountName": "张三",
      "accountNo": "003"
    },
    "accountDTOList": [
      {
        "balance": 3400,
        "account": "0002"
      },
      {
        "balance": 77800,
        "account": "0004"
      }
    ],
    "result1": "doService1() result"
  }
}
```
### 21 Java聚合服务调用Java延迟服务
通过Biz-SIP的开放API接口，在脚本服务中调用SAF服务，对server3服务进行多达5次的服务重试调用。
![](2cdb9c26523ad91e889db449a442f35d.svg)

上送请求数据说明：

- “maxRetryNum”为最大请求次数
- “result”为最后的响应为成功还是失败。
```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/sample/service4" -X POST --data '{"maxRetryNum":2,"result":"success"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "c16b63d02e844183811ac7cc01a4a046",
  "parentTraceId": null,
  "timestamp": 1630217962113,
  "data": {
    "maxRetryNum": 2,
    "result": "success"
  }
}
```
### 22 聚合服务匹配SpringBean服务
通过Biz-SIP的开放API接口，通过在“prefix-service.yml”中约定的服务前缀，调用所绑定对应类（或接口）的SpringBean所对应的方法。
![](c4f49701e5979c46fee698cf81670113.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/springbean" -X POST --data '{"methodName":"doService1","params":["003"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "a4efdb7ea392416eabd048c283535a1e",
  "parentTraceId": null,
  "timestamp": 1630398028746,
  "data": {
    "result": "doService1() result"
  }
}

$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/springbean" -X POST --data '{"methodName":"doService2","params":["003",1]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "0ac04237c79a415daa76ed6543dce839",
  "parentTraceId": null,
  "timestamp": 1630398312998,
  "data": {}
}

$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/springbean" -X POST --data '{"methodName":"queryCustomerDTO","params":["003"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "d20837e2b76048f8aabb09915d691f98",
  "parentTraceId": null,
  "timestamp": 1630398340509,
  "data": {
    "result": {
      "sex": "1",
      "isMarried": false,
      "customerId": "001",
      "name": "张三",
      "age": 20
    }
  }
}

```
### 23 Sink聚合服务直接调用Sink
通过Biz-SIP的开放API接口，通过配置的Sink聚合服务，直接调用CRM服务（用CrmServer.java类作为模拟），查询返回对应的账户名称。
![](ea627b4bd4f84a360a6ab7657e531530-20210924202843280.svg)

```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/sinkService" -X POST --data '{"accountNo":"003"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "9856ede98cf2492c9d17db5e86e53752",
  "parentTraceId": null,
  "timestamp": 1630474662509,
  "data": {
    "accountName": "张三",
    "accountNo": "003"
  }
}

```
### 24 聚合服务调用DelayServiceClient接口服务
通过Biz-SIP的开放API接口，在脚本服务中调用SAF服务，对server3服务进行多达5次的服务重试调用。
![](0d505c56488b51d0744b118feb884530.svg)

上送请求数据说明：

- “maxRetryNum”为最大请求次数
- “result”为最后的响应为成功还是失败。
```shell
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:/sample/service5" -X POST --data '{"method":"notify","maxRetryNum":3,"result":"success"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": null,
  "extMessage": null,
  "traceId": "63cd38f36fa647cf9be0403c81e2c058",
  "parentTraceId": null,
  "timestamp": 1630541890888,
  "data": {
    "method": "notify",
    "className": "com.bizmda.bizsip.sample.sink.api.SinkInterface1",
    "params": [
      3,
      "success"
    ]
  }
}
```
