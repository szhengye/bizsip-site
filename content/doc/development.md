---
title: 系统开发
date: '2021-09-24'
type: book
weight: 40
---

介绍使用Java代码来开发基于Biz-SIP中间件的系统。

<!--more-->

## 一、应用开发指南
Biz-SIP中间件在开发应用时，主要涉及配置文件编写和Java代码开发，系统架构如下所示：
![image.png](1632227276119-7bdc5979-8ddf-4498-84db-14fbbf44f203-20210924205613130.png)

上图中蓝底黑字所呈现的部分，就是需要Java代码开发的，涉及适配层、应用层和领域层。蓝色虚箭头线所涉及的接口，就是不同层次之间需要互相依赖的接口定义。

### 1. 渠道接入模块
渠道接入模块主要负责各种渠道的接入，主要功能包括：

- 渠道接入的安全处理，包括报文加解密、加签解签；
- 渠道接入的报文打包和解包；
- 渠道接入的报文校验；
- 渠道接入的交易处理，包括流水记录、风控处理、终端管理等；
- 调用应用层服务。
#### 1.1 报文打包和解包
平台提供JSON、XML、定长、有分隔符、ISO8583的格式转换器（converter），如果采用这些平台提供的converter，就需要约定Source ID。
如果采用编码实现报文打解包，就可以不考虑Source ID的约定。
#### 1.2 报文校验
在应用层中，有对应用层聚合服务的校验机制，包括域级校验和服务级校验，可以考虑统一在应用层进行报文检验。
但如果是仅针对于该特定渠道的报文校验，就只能考虑在渠道接入模块做了。
#### 1.3 交易处理
对于简单的交易处理，可以考虑用Spring Service来做交易处理。
在编码时注意，调用应用层聚合服务可能会导致时延较长，应避免在有数据库操作的Spring Service类中调用应用层聚合服务，并在有数据库操作的Spring Service类上尽量加@Transactional，以避免长时间锁表。
#### 1.4 调用应用层服务
统一采用“SourceClientFactory.getBizServiceClient(Class<T> tClass,String bizServiceId)”来调用应用层聚合服务。
对于Sink服务，在渠道接入模块不要直接调用（也无法直接调用），可以在应用层定义关联些Sink服务的sink-service，来透传调用。
### 2. 第三方系统接入模块
第三方系统接入，主要包括以下几种方式：

- 同步调用第三方系统服务：标准的sink服务，在领域层进行封装sink服务。
- 异步调用第三方系统服务：
   - 通知第三方系统服务：简单的就是标准的sink服务，如果涉及存储转发，则需要使用延迟服务来进行没有收到接收回执后的重发操作。
   - 第三方系统调用我方服务：这种类型请参照上节对于渠道接入模块的介绍，这里不再展开讲。

主要涉及的功能有：

- 接收Integrator发来的服务请求；
- 第三方接入的安全处理，包括报文加解密、加签解签；
- 第三方接入的报文打包和解包；
- 第三方接入的交易处理，包括流水记录、风控处理等；
- 异步回调现场恢复
- 通知服务的存储转发
- 事务完整性保障；
#### 2.1 接收Integrator发来的服务请求
Sink模块是通过connector接收Integrator发来的服务请求，connector主要有二种：

- bean：需要定义Interface接口类，一般在第三方渠道对接用得比较少；
- sink-bean：无需定义接口，根据需求也有二种类型的处理：
   - 采用Biz-SIP标准的JSONObject来做入参、出参，一般不涉及到报文格式转换或报文格式转换由开发者自己编码完成，JSONObjectSinkBeanInterface接口约定如下：
```java
public interface JSONObjectSinkBeanInterface {
    /**
     * JSONObject SinkBean服务调用接口
     * @param packMessage 传入的消息
     * @return 返回值
     * @throws BizException
     */
    public JSONObject process(JSONObject packMessage) throws BizException;
}
```

   - 采用字节流byte[]来做入参、出参，一般是涉及到不同消息类型的转换，SinkBeanInterface接口约定如下：
```java
public interface SinkBeanInterface {
    /**
     * Java服务调用接口
     * @param beforeJsonObject 打包前的JsonObject
     * @param packMessage 传入的消息
     * @return 返回值
     * @throws BizException
     */
    public byte[] process(JSONObject beforeJsonObject, byte[] packMessage) throws BizException;
}
```
#### 2.2 报文打包和解包
平台提供JSON、XML、定长、有分隔符的格式转换器（converter），如果采用这些平台提供的converter，就需要Sink实现SinkBeanInterface接口，传入参数是经过converter转换后的字节流byte[]。
如果由开发者自己编码实现报文打解包，建议Sink实现JSONObjectSinkBeanInterface接口，传入参数是平台约定的JSONObject。
#### 2.3 交易处理
对于简单的交易处理，可以考虑用SpringBean（@Service）来做交易处理。
在编码时注意，调用第三方应用时可能会导致时延较长，应避免在有数据库操作的SpringBean类中调用第三方应用服务，并在有数据库操作的SpringBean类上尽量加@Transactional，以避免长时间锁表。
另外，在实现SinkBeanInterface或JSONObjectSinkBeanInterface接口时，同时实现SinkBeanAspectInterface切面接口，也是一种简单的做法，并在切面方法中实现交易处理逻辑，并在方法上加@Transactional，以避免长时间锁表。
```java
public interface SinkBeanAspectInterface {
    /**
     *
     * @param jsonObject 一般为convertor处理前传入的JSONObject
     * @return 经过处理后的JSONObject
     */
    public JSONObject beforeProcess(JSONObject jsonObject);

    /**
     *
     * @param beforeJsonObject 一般为convertor处理前传入的JSONObject
     * @param afterJsonObject 经过process()处理后返回的JSONObject
     * @return
     */
    public JSONObject afterProcess(JSONObject beforeJsonObject,JSONObject afterJsonObject);

    /**
     *
     * @param jsonObject 传入process()的JSONObject
     * @param bizException 在process()中抛出的异常
     */
    public void handleProcessException(JSONObject jsonObject,BizException bizException);
}
```
#### 2.4 异步回调现场恢复
通过领域层sink服务传出消息，应用层服务（也称为异步调用的前半段服务）即完成；再通过渠道接入模块再回调后半段服务（一般是另一个应用层服务）,存在恢复交易现场的操作，可以调用SipService.loadAsyncContext()来实现，这和接入第三方时调用的SipService.saveAsyncContext()是相配套的。
#### 2.5 通知服务的存储转发
在应用层服务通过IntegratorClientFactory.getDelayBizServiceClient()发起延迟服务，实现通知服务的存储转发。
#### 2.6 事务完整性保障
在应用层服务通过IntegratorClientFactory.getDelayBizServiceClient()发起延迟服务，来实现补偿，包括向前补偿和向后补偿这二种方式。
### 3. 交易处理模块
交易处理模块就是一种特殊的Sink模块，是通过connector接收Integrator发来的服务请求，由于交易处理模块不涉及消息格式转换，所以connector主要有二种：

- bean：需要定义Interface接口类；
- sink-bean：采用Biz-SIP标准的JSONObject来做入参、出参，交易处理模块需要实现JSONObjectSinkBeanInterface接口。
#### 3.1 bean（Interface接口类交易处理）
采用bean类型connector，接口明确清晰，调用方一目了然，但需要调用双方提前约定Interface接口类。
在交易类型超过5个时，建议采用Command Executor（命令处理程序），分成多个xxxCmdExe类来进行处理，这些类应统一继承AbstractBeanCmdExe类。
#### 3.2 sink-bean（JSONObject出入参交易处理）
采用sink-bean类型connector，接口不用提前约定，扩展性强，但缺点也是很明显，调用双方缺乏接口约定的机制。
在交易类型超过5个时，建议采用Command Executor（命令处理程序），分成多个xxxCmdExe类来进行处理，这些类应统一继承AbstractSinkBeanCmdExe类，并统一在execute()中实现业务逻辑处理。
### 4. OpenAPI接口模块
OpenAPI接口模块主要功能：

- 敏感数据加解密及报文加签验签；
- RESTful接口封装；
- OpenAPI接口文档的生成；
- Sandbox；
- 聚合服务的调用
#### 4.1 RESTful接口封装和OpenAPI接口文档
根据接口要求开发controller类，并加上API注释，自动生成Swagger/Knife4j文档。
#### 4.2 聚合服务的调用
统一采用“SourceClientFactory.getBizServiceClient(Class<T> tClass,String bizServiceId)”来调用应用层聚合服务。
### 5. 应用层服务
应用层服务功能：

- 领域层服务（Sink服务）的编排
- 领域层服务（Sink服务）透传到适配层服务（Source服务和OpenAPI）
- 存储转发服务的封装
- 补偿交易的封装

应用层服务，主要包括以下三类服务：

- 通过sink-service透传的领域层sink服务：服务命名为”sink/xxx“，其中xxx和领域层服务xxx-sink前缀一致；
- 只涉及某一个sink的延迟服务：服务命名为”sink/xxx-delay/...“，其中xxx和领域层服务xxx-sink前缀一致，但涉及到多个sink聚合编排的延迟服务，应直接放在相关的聚合应用服务中；
- 聚合应用服务：服务命名为”app/yyy/...“，其中yyy是聚合应用服务名，涉及到多个sink聚合编排的延迟服务，也放在这里。
### 6. 项目依赖
#### 6.1 适配层source模块
在pom.xml文件中添加source-spring-boot-starter依赖，以及应用层app模块和领域层sink模块相关联的client接口相关包：
```xml
        <dependency>
            <groupId>com.bizmda.bizsip</groupId>
            <artifactId>source-spring-boot-starter</artifactId>
          	<version>1.0-SNAPSHOT</version>
        </dependency>
```
#### 6.2 应用层app模块
在pom.xml文件中添加integrator-spring-boot-starter依赖，以及领域层sink模块相关联的client接口相关包：
```xml
        <dependency>
            <groupId>com.bizmda.bizsip</groupId>
            <artifactId>integrator-spring-boot-starter</artifactId>
          	<version>1.0-SNAPSHOT</version>
        </dependency>
```
#### 6.3 领域层sink模块
在pom.xml文件中添加sink-spring-boot-starter依赖：
```xml
        <dependency>
            <groupId>com.bizmda.bizsip</groupId>
            <artifactId>sink-spring-boot-starter</artifactId>
          	<version>1.0-SNAPSHOT</version>
        </dependency>
```
### 7. 各模块application.yml配置
#### 7.1 网关模块
Open API网关只在Spring Cloud版本中存在，在Spring Boot版本中不需要部署。
api-gateway模块中的application.yml：

| 配置项 | 配置说明 |
| --- | --- |
| server.port | 为api-gateway网关的侦听端口，建议用8000 |
| spring.cloud.nacos.discovery.server-addr | Nacos服务端口 |
| spring.cloud.gateway.routes | 为配置转发给Integrator服务整合器的路由断言，建议不要修改。 |

例子：
```yaml
server:
  port: 8000

spring:
  application:
    name: bizsip-api-gaetway

  cloud:
    nacos:
      discovery:
        server-addr: bizsip-nacos:8848

    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      routes:
        - id: bizsip-integrator
          uri: lb://bizsip-integrator
          predicates:
            - Path=/api/**

logging:
  level:
    com.bizmda.bizsip: debug

graceful:
  shutdown:
    enabled: true
```
#### 7.2 适配层source模块
Source模块的application.yml中的主要相关参数：

| 配置项 | 配置说明 |
| --- | --- |
| bizsip.config-path | Biz-SIP中间件配置文件根目录位置 |
| bizsip.integrator-url | 服务聚合器Integrator的接口url地址 |

例子：
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
    com.bizmda.bizsip: debug

```
#### 7.3 应用层app模块
应用层app Integrator模块中的application.yml中的主要相关参数：

| 配置项 | 配置说明 |
| --- | --- |
| bizsip.config-path | Biz-SIP中间件配置文件根目录位置 |
| server.port | 服务整合器（Integrator）的微服务端口，建议用8888端口，避免和其它端口相冲突 |
| spring.cloud.nacos.discovery.server-addr | Nacos服务端口 |
| spring.datasource.* | 数据库连接配置（用于服务脚本中db对象） |
| spring.redis.* | redis连接配置（用于服务脚本中redis对象） |
| spring.rabbitmq.* | RabbitMQ配置（用于事务管理器） |

例子：
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
    com.bizmda.bizsip: debug

```
#### 7.4 领域层sink模块
领域层Sink应用的application.yml中的主要相关参数：

| 配置项 | 配置说明 |
| --- | --- |
| bizsip.config-path | Biz-SIP中间件配置文件根目录位置 |
| bizsip.sink-id（可选） | sink-spring-boot-starter中自带通用SinkController对应的sink-id参数。
只有启用SinkController才会配置该参数 |

SinkController为通用的Sink处理模块，能接收来自Integrator发来服务，并根据sink.yml中Sink的配置自动进行相应处理，如果没有特殊的处理需求，一般建议直接采用SinkController即可。
要启用SinkController，需要在应用启动类中，加上对应的Spring类扫描路径（"com.bizmda.bizsip.sink.controller"）​：
```java
@SpringBootApplication
@MapperScan("com.xbank.infrastructure.db.customer.mapper")
@ComponentScan(basePackages={"cn.hutool.extra.spring",
        "com.bizmda.bizsip.sink.controller", "com.xbank"})
@Import(cn.hutool.extra.spring.SpringUtil.class)
public class CustomerSinkApplication {
    public static void main(String[] args) {
        SpringApplication.run(CustomerSinkApplication.class, args);
    }
}
```
例子：
```yaml
server:
  port: 8001
spring:
  application:
    name: customer-sink

  cloud:
    nacos:
      discovery:
        server-addr: bizsip-nacos:8848

  datasource:
    url: jdbc:mysql://bizsip-mysql/xbank?autoReconnect=true
    username: root
    password: bizsip123456
    driver-class-name: com.mysql.cj.jdbc.Driver

bizsip:
  config-path: /Users/shizhengye/IdeaProjects/xbank/xbank-app/config
  sink-id: customer-sink

logging:
  level:
    com.bizmda.bizsip: debug
    com.xbank: trace
```
### 8. 命名规则
| 用途 | 规范 | 解释 |
| --- | --- | --- |
| 领域层服务（sink） | xxx-sink |  |
| 应用层服务（service） | sink/xxx | 通过sink-service透传到领域层的sink服务 |
|  | sink/xxx-delay/... | 某一个sink服务相关的延迟服务，这里的延迟服务只涉及到该sink的调用，不涉及其它sink服务的调用。 |
|  | app/yyy/... | 封装的某一应用层服务集合，包括一系列相关应用层服务，涉及到多个sink服务的延迟服务，也放在这里。 |
| 适配层服务（source） | xxx-source | 涉及到消息格式转换配置时，才需要配置Source ID |

## 二、调用Sink服务和App服务
### 1. Source调用App服务


统一采用“SourceClientFactory.getBizServiceClient(Class<T> tClass,String bizServiceId)”来调用应用层聚合服务（tClass必须是接口类），二种调用方式：

- 接口约定调用：由应用层定义接口调用的Interface接口类，由渠道接入模块所引用。如下例所示：
```java
@RestController
@RequestMapping("/personal")
public class PersonalController  {
    private PersonalAppInterface personalAppInterface = SourceClientFactory
            .getBizServiceClient(PersonalAppInterface.class,"app/personal");
	...

    @GetMapping(value ="/getCustomerAndAccountList")
    public CustomerAndAccountList getCustomerAndAccountList(String customerId) {
        return this.personalAppInterface.getCustomerAndAccountList(customerId);
    }

    @GetMapping(value ="/getAccountListByCustomerId")
    public List<Account> getAccountListByCustomerId(String customerId) {
        return this.personalAppInterface.getAccountListByCustomerId(customerId);
    }

    @GetMapping(value ="/getCustomer")
    public Customer getCustomer(String customerId) {
        return this.personalAppInterface.getCustomer(customerId);
    }
	...
}

```

- 非接口约定调用：采用平台通用JSONObject类型，Interface接口类统一采用BizMessageInterface接口类，并统一用call()来调用应用层聚合服务。如下例所示：
```java
@RestController
@RequestMapping("/personal")
public class PersonalController  {
    private BizMessageInterface payment1SinkInterface = SourceClientFactory
            .getBizServiceClient(BizMessageInterface.class,"sink/payment1");
    ...
        
    @GetMapping(value ="/send2Payment")
    public BizMessage<JSONObject> send2Payment(String message) throws BizException {
        JSONObject jsonObject = new JSONObject();
        jsonObject.set("message",message);
        return this.payment1SinkInterface.call(jsonObject);
    }
}
```
### 2. App调用Sink服务
统一采用“IntegratorClientFactory.getSinkClient(Class<T> tClass,String sinkId)”来调用领域层Sink服务（tClass必须是接口类），有二种调用方式：

- 接口约定调用：由应用层定义接口调用的Interface接口类，由渠道接入模块所引用。如下例所示：
```java
@Service
public class PersonalAppService implements PersonalAppInterface {
    private AccountSinkInterface accountSinkInterface = IntegratorClientFactory
            .getSinkClient(AccountSinkInterface.class,"account-sink");
    private CustomerSinkInterface customerSinkInterface = IntegratorClientFactory
            .getSinkClient(CustomerSinkInterface.class,"customer-sink");
	...
        
    @Override
    public CustomerAndAccountList getCustomerAndAccountList(String customerId) {
        Customer customer = this.customerSinkInterface.getCustomer(customerId);
        List<Account> accountList = this.accountSinkInterface.getAccountListByCustomerId(customerId);
        CustomerAndAccountList customerAndAccountList = new CustomerAndAccountList();
        customerAndAccountList.setCustomer(customer);
        customerAndAccountList.setAccountList(accountList);
        return customerAndAccountList;
    }
	...
}
```

- 非接口约定调用：采用平台通用JSONObject类型，Interface接口类统一采用BizMessageInterface接口类，并统一用call()来调用应用层聚合服务。如下例所示：
```java
@Service
public class PersonalAppService implements PersonalAppInterface {
	...
    private BizMessageInterface payment1SinkInterface = IntegratorClientFactory
            .getSinkClient(BizMessageInterface.class,"payment1-sink");
    private BizMessageInterface payment2SinkInterface = IntegratorClientFactory
            .getSinkClient(BizMessageInterface.class,"payment2-sink");
	...
    @Override
    public BizMessage<JSONObject> send2Payment1(Object message) throws BizException {
        JSONObject jsonObject = new JSONObject();
        jsonObject.set("message",message);
        return this.payment1SinkInterface.call(jsonObject);
    }

    @Override
    public BizMessage send2Payment2(String tranMode, String tranCode, Object message) throws BizException {
        JSONObject jsonObject = new JSONObject();
        jsonObject.set("tranCode",tranCode);
        jsonObject.set("tranMode",tranMode);
        jsonObject.set("message",message);
        return this.payment2SinkInterface.call(jsonObject);
    }
    ...
}
```
### 3. App调用延迟服务
延迟服务也是位于应用层的App服务，延迟服务只能由应用层的App服务来进行嵌套调用，不能从适配层直接调用延迟服务。
App调用延迟服务，统一采用“IntegratorClientFactory.getDelayBizServiceClient(Class<T> tClass, String bizServiceId, int... delayMilliseconds)”来调用（tClass必须是接口类），同样有接口约定调用和非接口约定调用二种方式，其中非接口约定调用是采用BizMessageInterface接口类，并统一用call()来调用应用层聚合服务，如下例所示：
```java
@Service
public class PersonalAppService implements PersonalAppInterface {
    private AccountSinkInterface accountSinkInterface = IntegratorClientFactory
            .getSinkClient(AccountSinkInterface.class,"account-sink");
    private CustomerSinkInterface customerSinkInterface = IntegratorClientFactory
            .getSinkClient(CustomerSinkInterface.class,"customer-sink");
    private BizMessageInterface payment1SinkInterface = IntegratorClientFactory
            .getSinkClient(BizMessageInterface.class,"payment1-sink");
    private BizMessageInterface payment2SinkInterface = IntegratorClientFactory
            .getSinkClient(BizMessageInterface.class,"payment2-sink");
    private PersonalAppInterface personalAppDelayInterface = IntegratorClientFactory
            .getDelayBizServiceClient(PersonalAppInterface.class,"app/personal",
                    0,1000,2000,4000,8000,16000,32000);
	...
    @Override
    public void payoutForward(String tranMode,String accountId, long amount) throws BizException {
        log.info("account出金:{},{}",accountId,amount);
        this.accountSinkInterface.payout(accountId,amount);
        JSONObject jsonObject = new JSONObject();
        jsonObject.set("tranCode","pay");
        jsonObject.set("tranMode",tranMode);
        jsonObject.set("accountId",accountId);
        jsonObject.set("tranAmount",amount);
        BizMessage<JSONObject> bizMessage = null;
        try {
            log.info("payment缴费...");
            bizMessage = this.payment2SinkInterface.call(jsonObject);
        } catch (BizException e) {
            if (e.isTimeOutException()) {
                log.info("payment交易超时,开始payout补偿...");
                this.personalAppDelayInterface.payoutForwardCompensate(jsonObject);
                return;
            }
            else {
                throw e;
            }
        }
        log.info("payment缴费成功!");
        log.info("payout成功!");
    }
}
```
## 三、开发Sink服务
通过Java来开发的Sink服务，主要支持以下二类：sink-bean类型、bean类型。
### 1. sink-bean类sink服务
sink-bean主要有以下特点：

- sink接口为标准的BizMessage<JSONObject>类型即可，其它没有格式约定；
- 在sink.yml中配置”sink-bean“类型的connector，服务类支持JavaBean和SpringBean二种挂接方式；

sink-bean的实现，应基于<SinkBeanInterface>或<JSONObjectSinkBeanInterface>接口开发服务类，实现process()方法，2个接口区别在于出入参数的类型。
#### 1.1 实现<JSONObjectSinkBeanInterface>接口的sink服务
JSONObjectSinkBeanInterface接口中的process()出入参参数采用Biz-SIP标准的JSONObject：这种类型的sink服务一般不涉及到报文格式转换或报文格式转换由开发者自己编码完成，JSONObjectSinkBeanInterface接口约定如下：
```java
public interface JSONObjectSinkBeanInterface {
    /**
     * JSONObject SinkBean服务调用接口
     * @param packMessage 传入的消息
     * @return 返回值
     * @throws BizException
     */
    public JSONObject process(JSONObject packMessage) throws BizException;
}
```
举例如下：
```java
@Service
public class Payment2SinkService implements JSONObjectSinkBeanInterface {
    @Autowired
    private TimeoutCmdExe timeoutCmdExe;
    @Autowired
    private TimeoutAndFailCmdExe timeoutAndFailCmdExe;
    @Autowired
    private TimeoutAndSuccessCmdExe timeoutAndSuccessCmdExe;
    @Autowired
    private SuccessCmdExe successCmdExe;

    @Override
    public JSONObject process(JSONObject jsonObject) throws BizException {
        log.info("传入消息:\n{}", jsonObject.toString());
        AbstractSinkBeanCmdExe sinkBeanCmdExe;
        String tranMode = (String)jsonObject.get("tranMode");
        switch (tranMode) {
            case "timeout":
                // 收到交易后，永远返回超时
                return timeoutCmdExe.execute(jsonObject);
            case "3timeout-fail":
                // 收到交易后，前3次返回超时，第4次返回失败码
                return timeoutAndFailCmdExe.execute(jsonObject);
            case "3timeout-success":
                // 收到交易后，前3次返回超时，第4次成功返回原报文
                return timeoutAndSuccessCmdExe.execute(jsonObject);
            default:
                //其它情况,成功返回原报文
                return successCmdExe.execute(jsonObject);
        }
    }
}
```
#### 1.2 实现<SinkBeanInterface>接口的sink服务
SinkBeanInterface接口的process()出入参采用字节流byte[]：这种类型的sink服务一般是涉及到不同消息类型的转换，在sink.yml中支持多种convert，传入process()方法的参数为经过convert转换后的消息。SinkBeanInterface接口约定如下：
```java
public interface SinkBeanInterface {
    /**
     * Java服务调用接口
     * @param beforeJsonObject 打包前的JsonObject
     * @param packMessage 传入的消息
     * @return 返回值
     * @throws BizException
     */
    public byte[] process(JSONObject beforeJsonObject, byte[] packMessage) throws BizException;
}
```
在process()方法中传入的beforeJsonObject参数，是传入sink但还没经过打包的JSONObject对象，packMessage参数已经根据sink的消息格式配置，完成了对于目标消息格式的转换。
举例如下：
```java
@Service
public class Payment1SinkService implements SinkBeanInterface {
    @Override
    public byte[] process(JSONObject beforeJsonObject, byte[] inMessage) throws BizException {
        log.info("传入消息:\n{}", BizUtils.buildHexLog(inMessage));
        return inMessage;
    }
}
```
#### 1.3 实现<SinkBeanAspectInterface>切面接口
在实现SinkBeanInterface或JSONObjectSinkBeanInterface接口时，可以通过实现SinkBeanAspectInterface切面接口，能在convertor处理前、处理后以及处理异常时实现注入逻辑，接口约定如下：
```java
public interface SinkBeanAspectInterface {
    /**
     *
     * @param jsonObject 一般为convertor处理前传入的JSONObject
     * @return 经过处理后的JSONObject
     */
    public JSONObject beforeProcess(JSONObject jsonObject);

    /**
     *
     * @param beforeJsonObject 一般为convertor处理前传入的JSONObject
     * @param afterJsonObject 经过process()处理后返回的JSONObject
     * @return
     */
    public JSONObject afterProcess(JSONObject beforeJsonObject,JSONObject afterJsonObject);

    /**
     *
     * @param jsonObject 传入process()的JSONObject
     * @param bizException 在process()中抛出的异常
     */
    public void handleProcessException(JSONObject jsonObject,BizException bizException);
}
```
### 2. bean类sink服务
bean类sink服务有以下特点：

- sink接口为BizMessage<JSONObject>类型，JSONObject有格式要求，消息体中应包括：className（可选）、methodName（可选）、params（必选），params为JSONArray类型；
- 可自由开发服务类；
- 在sink.yml中配置”bean“类型的connector，服务类支持JavaBean和SpringBean二种挂接方式；
- 在sink.yml中convert只支持simple-json类型（后期可能会取消对于bean类sink服务的convert配置）。
- bean类在开发时，一定要先定义一个Interface类，这个Interface类会被调用方（App应用层）所引用。

bean类sink服务，如下例所示：
```java
@Service
public class AccountSinkService implements AccountSinkInterface {
    @Autowired
    private PayoutCmdExe payoutCmdExe;
    @Autowired
    private  PayoutCompensationCmdExe payoutCompensationCmdExe;
    @Autowired
    private GetAccountListByCustomerIdCmdExe getAccountListByCustomerIdCmdExe;

    @Override
    public List<Account> getAccountListByCustomerId(String customerId) {
        return this.getAccountListByCustomerIdCmdExe.getAccountListByCustomerId(customerId);
    }

    @Override
    public Account payout(String accountId, long amount) {
        return this.payoutCmdExe.payout(accountId,amount);
    }

    @Override
    public Account payoutCompensation(String accountId, long amount) {
        return this.payoutCompensationCmdExe.payoutCompensation(accountId,amount);
    }
}
```
## 四、开发app服务
通过Java来开发的app服务，主要支持以下二类：integrator-bean-service类型、bean-service类型。
### 1. integrator-bean-service类app服务
integrator-bean-service主要有以下特点：

- sink接口为标准的BizMessage<JSONObject>类型即可，其它没有格式约定；
- 在service.yml中配置type为”integrator-bean-service“，服务类只支持SpringBean的挂接方式；

integrator-bean-service的实现，应基于<IntegratorBeanInterface>接口开发服务类，实现doBizService()方法。
IntegratorBeanInterface接口约定如下：
```java
public interface IntegratorBeanInterface {
    /**
     * 执行聚合服务
     * @param message 传入的消息
     * @return 返回的消息
     */
    public abstract BizMessage<JSONObject> doBizService(BizMessage<JSONObject> message) throws BizException;
}
```
举例如下：
```java
@Service
public class IntegratorService3 implements IntegratorBeanInterface {
    private SinkInterface1 sinkInterface1 = IntegratorClientFactory
            .getSinkClient(SinkInterface1.class,"sink12");
    private BizMessageInterface sinkInterface2 = IntegratorClientFactory
            .getSinkClient(BizMessageInterface.class,"sink1");

    @Override
    public BizMessage<JSONObject> doBizService(BizMessage<JSONObject> message) throws BizException {
        String result1 = this.sinkInterface1.doService1("001");
        this.sinkInterface1.doService2("002",3);
        AccountDTO[] accountDTOs = this.sinkInterface1.queryAccounts(AccountDTO.builder().account("002").build());
        List<AccountDTO> accountDTOList = this.sinkInterface1.queryAccountList(AccountDTO.builder().account("002").build());
        CustomerDTO customerDTO = this.sinkInterface1.queryCustomerDTO("001");
        JSONObject jsonObject1 = new JSONObject();
        jsonObject1.set("accountNo","003");
        BizMessage<JSONObject> bizMessage = this.sinkInterface2.call(jsonObject1);
        JSONObject jsonObject = new JSONObject();
        jsonObject.set("result1",result1);
        jsonObject.set("accountDTOs",accountDTOs);
        jsonObject.set("accountDTOList",accountDTOList);
        jsonObject.set("customerDTO",customerDTO);
        jsonObject.set("sink1",bizMessage.getData());
        return BizMessage.buildSuccessMessage(message,jsonObject);
    }
}
```
### 2. bean-service类app服务
bean-service类app服务有以下特点：

- app接口虽然都要求为BizMessage<JSONObject>类型，但是对JSONObject有格式要求，消息体中应包括：className（可选）、methodName（可选）、params（必选），params为JSONArray类型；
- 可自由开发服务类，但上送JSONObject的className、methodName、params应该和类接口相匹配；
- 在service.yml中配置”bean-service“类型的聚合服务，服务类只支持SpringBean来进行挂接；
- bean-service类在开发时，一定要先定义一个Interface类，这个Interface类会被调用方（一般是渠道层source，app服务如作为延迟服务会被应用层调用）所引用。

bean-service类app服务，如下例所示：
```java
@Service
public class SinkClientInterface1Impl implements SinkInterface1 {
    private SinkInterface1 sinkInterface1 =  IntegratorClientFactory.getSinkClient(SinkInterface1.class,"sink12");;
    int retryNum = 0;

    @Override
    public String doService1(String arg1) {
        return this.sinkInterface1.doService1(arg1);
    }

    @Override
    public void doService2(String arg1, int arg2) {
        this.sinkInterface1.doService2(arg1,arg2);
    }

    @Override
    public CustomerDTO queryCustomerDTO(String customerId) {
        return this.sinkInterface1.queryCustomerDTO(customerId);
    }

    @Override
    public AccountDTO[] queryAccounts(AccountDTO accountDTO) {
        return this.sinkInterface1.queryAccounts(accountDTO);
    }

    @Override
    public List<AccountDTO> queryAccountList(AccountDTO accountDTO) {
        return this.queryAccountList(accountDTO);
    }
	...
}
```
