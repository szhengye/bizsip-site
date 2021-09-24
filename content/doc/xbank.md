---
title: xbank示例项目
date: '2021-09-24'
type: book
weight: 80
---

通过xbank项目来介绍用Biz-SIP中间件开发一个系统的全流程

<!--more-->

**xbank项目版本库：**[**https://gitee.com/szhengye/xbank.git**](https://gitee.com/szhengye/xbank.git)

## 一、Biz-SIP架构和DDD领域驱动设计入门介绍
### 1. DDD领域驱动设计
2004年，Eric Evans完成了《Domain-Driven Design Tackling Complexity in the Heart of Software》一书，提出了一套针对业务领域建模的方法论和思想--领域驱动设计，简称DDD。
领域驱动设计（DDD）是一种基于模型驱动的软件设计方式。它以领域为核心，分析领域中的问题，通过建立一个领域模型来有效地解决领域中的核心、复杂问题。
领域驱动设计提出了一套核心构造块（Building Blocks），如聚合、实体、值对象、领域服务、领域工厂、仓储、领域事件等。这些构造块是对面向对象领域建模的一些核心最佳实践的浓缩。这些构造块可以使得我们的设计更加标准、有序。
利用 DDD 提倡的分层、六边形等架构，分离了业务复杂度和技术复杂度，使得系统具备更强的扩展性和弹性；战术层面提供了元模型（聚合，实体，值对象，服务，工厂，仓储等等）帮助构建清晰、稳定，能快速响应变化和新需求能力的应用。
### 2. COLA架构
COLA 4.0 推荐的应用架构如下图所示：
![image.png](1630415427800-07efac34-8cd6-4df1-82d2-d19af10c513c.png)
1）适配层（Adapter Layer）：负责对前端展示（web，wireless，wap）的路由和适配，对于传统B/S系统而言，adapter就相当于MVC中的controller；
2）应用层（Application Layer）：主要负责获取输入，组装上下文，参数校验，调用领域层做业务处理，如果需要的话，发送消息通知等。层次是开放的，应用层也可以绕过领域层，直接访问基础实施层；
3）领域层（Domain Layer）：主要是封装了核心业务逻辑，并通过领域服务（Domain Service）和领域对象（Domain Entity）的方法对App层提供业务实体和业务逻辑计算。领域是应用的核心，不依赖任何其他层次；
4）基础设施层（Infrastructure Layer)：主要负责技术细节问题的处理，比如数据库的CRUD、搜索引擎、文件系统、分布式服务的RPC等。此外，领域防腐的重任也落在这里，外部依赖需要通过gateway的转义处理，才能被上面的App层和Domain层使用。

---

## 二、体验Biz-SIP示例项目xbank
### 1. 示例项目xbank简介
xbank是一家商业银行，面向个人客户和公司客户，其中个人客户业务包括存款、贷款、缴费等业务；银行业务渠道除了传统柜面以外，还有网上银行、手机银行、ATM、POS等，最近准备上一个针对银行合作伙伴的基于OPENAPI网关的开放平台渠道。
本示例项目是以个人客户中的存款查询和缴费业务为例子，渠道采用OPENAPI开放接口，后台系统对接个人客户存款系统和个人客户信息系统，第三方对接缴费平台，来演示如何打造基于Biz-SIP中间件的银行业务中台。
![image.png](1630736586701-bb5dd0f9-f227-46c9-811f-8aa91d2e5ec4.png)

按DDD分层架构，应用架构图如下所示：
![image.png](1631769310897-aa802b0c-0946-4bae-802c-525ff8d5d667.png)

### 3. xbank示例项目模板介绍
打开示例项目，如下图：
![image.png](1631770657971-e46b2907-ad55-40cf-a17c-4fc559de6440.png)
xbank项目分为以下模块：

- xbank-source：适配层模块，存放外部适配接入的适配层实现。
   - xbank-openapi-source：OPENAPI接入的适配层子模块
- xbank-app：应用层模块，存放应用层的服务实现。
- xbank-sink：领域层模块，存放按领域划分的领域服务实现。
   - xbank-account-sink：账户域子模块
   - xbank-customer-sink：客户域子模块
   - xbank-payment1-sink：缴费域子模块（存储转发）
   - xbank-payment1-sink：缴费域子模块（交易补偿）
- xbank-infrastructure：基础设施层模块，存放对数据库、内容存储、HSM等基础设施的访问能力实现。
   - xbank-account-db：账户数据库访问子模块
   - xbank-customre-db：客户数据库访问子模块
- xbank-client：接口模块，存放各层之间的服务接口，以及相关数据DTO。
   - xbank-account-sink-client：账户域服务接口子模块
   - xbank-customer-sink-client：客户域服务接口子模块
   - xbank-app-client：应用服务接口子模块
### 4. 创建数据库
执行项目中xbank-infrastructure/xbank-sql/xbank.sql脚本，以建立xbank演示库。
### 5. 启动xbank项目

- 启动应用层服务整合器
   - 执行xbank-app模块中的XbankAppApplication
- 启动领域层领域服务
   - 执行xbank-customer-sink子模块中的CustomerSinkApplication
   - 执行xbank-account-sink子模块中的AccountSinkAppliction
- 启动适配层OPENAPI接入服务
   - 执行xbank-openapi-source子模块中的OpenapiSourceApplication
### 4. 测试接口
#### 4.1 查询客户信息
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:sink/customer" -X POST --data '{"methodName":"getCustomer","params":["001"]}' http://localhost:8888/api|jq              
{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "2d3ed69623f8480a9d95bcd3ca03a4d3",
  "parentTraceId": null,
  "timestamp": 1630677591308,
  "data": {
    "result": {
      "sex": "1",
      "customerName": "张三",
      "customerId": "001",
      "age": 30
    }
  }
}
```
#### 4.2 查询账户列表
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:sink/account" -X POST --data '{"methodName":"getAccountListByCustomerId","params":["001"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "748b76566a5e436e9c2990cf9776789f",
  "parentTraceId": null,
  "timestamp": 1630677747532,
  "data": {
    "result": [
      {
        "accountId": "0001",
        "balance": 100,
        "customerId": "001"
      },
      {
        "accountId": "0002",
        "balance": 200,
        "customerId": "001"
      }
    ]
  }
}

```
#### 4.3 查询客户信息和账户列表
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:app/personal" -X POST --data '{"methodName":"getCustomerAndAccountList","params":["001"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "4fa2f86c23ce4b24abb1b0d23833c14f",
  "parentTraceId": null,
  "timestamp": 1630677780246,
  "data": {
    "result": {
      "accountList": [
        {
          "accountId": "0001",
          "balance": 100,
          "customerId": "001"
        },
        {
          "accountId": "0002",
          "balance": 200,
          "customerId": "001"
        }
      ],
      "customer": {
        "sex": "1",
        "customerName": "张三",
        "customerId": "001",
        "age": 30
      }
    }
  }
}
```
### 

## 三、项目实践：客户域服务的开发
### 1. 创建数据库
执行项目中xbank-infrastructure/xbank-sql/xbank.sql脚本，以建立xbank演示库。
### 2. 自动生成数据访问层代码
数据库创建后，用MybatisX插件自动生成数据访问层代码：
![image.png](1630738820784-7473f0a7-7ac5-4738-8a3e-68b5e128366d.png)

### 3. Customer领域服务接口和实现类的开发
先在xbank-customer-domain-client子模块中约定customer领域服务接口，这个接口是共享给领域层和应用层的：
![image.png](1631770763773-37715fd7-dd91-4faf-aab8-08fc5b98bd1d.png)
然后在xbank-customer-sink子模块中完成以下工作：

1. 基于此接口完成CustomerSinkService类的编写；
1. 完成CustomerSinkApplication启动类的编写；
1. 配置application.yml相关文件。

![image.png](1631769805722-f5fcc556-00c8-421c-85f3-ed62706f3870.png)
以上就完成了客户领域服务微服务的开发，接下来需要把这个客户领域服务微服务，通过sink接入Biz-SIP平台，配置xbank-app/config/sink.yml，就把CustomerSinkService类挂接到了sink（customer-sink)上：
![image.png](1631769899833-9983d1d9-936c-44b7-86d8-f98a9ee9e96d.png)

### 4. Customer领域服务的快速发布
Customer领域服务接入Biz-SIP平台后，能实现在Biz-SIP开放平台接口的快速发布，这需要在xbank-app/config/service.yml中配置一个类型为“sink-service”的聚合服务：
![image.png](1631770202721-005d08b1-92a5-4855-873b-e5ecd96c0802.png)
注：这类“sink-service”的聚合服务，命名规范为“sink/xxx”，xxx为sink名称。
启动Biz-SIP平台和Customer领域服务，就可以直接在Biz-SIP开放平台接口进行访问：

```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:sink/customer" -X POST --data '{"methodName":"getCustomer","params":["001"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "e763a42f9a2a49518c2bc6c157d08bab",
  "parentTraceId": null,
  "timestamp": 1630740781540,
  "data": {
    "result": {
      "sex": "1",
      "customerName": "张三",
      "customerId": "001",
      "age": 30
    }
  }
}
```
在上传的数据体中，“methodName”为方法名，“params”为输入参数。
### 5. Customer领域服务在应用层和适配层的定制
通过在service.yml中配置sink-service聚合服务，能实现已经挂接到Sink的领域服务的快速发布；但是，客户针对应用层和适配层，还是有个性化定制要求，这就涉及到应用层和适配层的定制。
同样以已经挂接到customer-sink上的Customer领域服务为例，先在xbank-app-client子模块中约定应用层接口，这个接口是共享给应用层和适配层的：
![image.png](1631784835222-e08c9e4d-4a53-4af3-b2a0-76ab3a1ec1ca.png)
然后在xbank-app模块中基于此接口完成PersonalAppService类的编写：
![image.png](1631770955734-46595cd1-5d39-4a62-9549-f69f3fbc6fa3.png)
在xbank-app/config/service.yml中配置一个类型为“bean-service”的聚合服务，并类名指定为上面实现的PersonalAppService应用层服务类：
![image.png](1631771374770-e4f66443-70d6-419e-958c-86009f30f6ff.png)
最后在适配层中，定制一个controller来对外发布服务：
![image.png](1631771757331-ffa6ca09-dffa-4faa-92e5-7e2d31d395af.png)
开发者可以在适配层的controller类中通过personalAppInterface，直接调用应用层的PersonalAppService服务：

```bash
$ curl http://localhost:9001/personal/getCustomer\?customerId=001|jq

{
  "customerId": "001",
  "customerName": "张三",
  "age": 30,
  "sex": "1"
}

```
## 四、项目实践：账户域服务的开发
### 1. Account领域服务的封装
Account领域服务是和Customer领域服务并列的，Account领域服务的封装，依次有以下步骤：

- 第1步：领域服务接口的约定：在xbank-account-sink-client中编写AccountSinkInterface接口；
- 第2步：领域服务的实现：创建xbank-account-sink子模块，基于第1步约定的接口，实现AccountSinkService SpringBean，并封装成能独立运行的微服务应用；
- 第3步：配置sink.yml文件，把AccountSinkService SpringBean作为Sink连接到Biz-SIP平台；

另外，Account领域服务在封装时，涉及到多个接口调用，对每个接口采用命令执行器（Command Executor），所有的接口实现继承AbstractBeanCmdExe类来实现：
![image.png](1631772314042-262db11c-3554-47c5-b1da-2bfed319f37d.png)

### 2. Account服务在适配层和应用层的开发和对外暴露
把sink服务，暴露给适配层接口调用，有2种方案：
> 方案一：

通过sink-service类型，打通适配层和应用层：直接把account领域服务所对应的sink，直接暴露给Biz-SIP开放平台接口。

- 第4步：配置service.yml文件，把account-sink直接作为sink-service暴露给Biz-SIP开放平台接口。

接口测试如下：
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:sink/account" -X POST --data '{"methodName":"getAccountListByCustomerId","params":["001"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "2280472dd72c4ee4a2b7214604e9cf27",
  "parentTraceId": null,
  "timestamp": 1630751382623,
  "data": {
    "result": [
      {
        "accountId": "0001",
        "balance": 100,
        "customerId": "001"
      },
      {
        "accountId": "0002",
        "balance": 200,
        "customerId": "001"
      }
    ]
  }
}

```
> 方案二：

通过bean-service，打通适配层和应用层：

- 第4步：应用服务接口的约定：在xbank-personal-app-client中编写PersonalAppInterface接口和CustomerAndAccountList DTO类；
- 第5步：应用服务的实现：在xbank-app模块中，基于第4步约定的接口实现PersonalAppService服务；
- 第6步：配置service.yml文件，把第5步开发PersonalAppService服务，通过bean-service的account-sink暴露给适配层；
- 第7步：在适配层创建xbank-openapi-source子模块，开发PersonalController类，暴露个性化的REST接口，并通过SourceClientFactory.getBizServiceClient(PersonalAppInterface.class,"app/personal")获取应用层服务的调用接口（具体请参见[链接](#lazZY)），在适配层中调用应用层约定接口。

接口测试如下：
```bash
$ curl http://localhost:9001/personal/getAccountListByCustomerId\?customerId=001|jq

[
  {
    "accountId": "0001",
    "balance": 100,
    "customerId": "001"
  },
  {
    "accountId": "0002",
    "balance": 200,
    "customerId": "001"
  }
]

```
## 五、项目实践：客户域和账户域服务在应用层的编排
上面已经分别实现了客户域和账户域服务的开发和部署，在应用层中，能方便地对域服务进行服务编排。
在xbank-app模块中，我们可以在PersonalAppService类中，方便地进行服务编排，代码如下：
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
在getCustomerAndAccountList()方法中，我们可以同时调用客户域的接口customerSinkInterface，也可以调用账户域的接口accountSinkInterface，从而实现多个域服务的混合编排。
这个聚合服务通过bean-service进行部署：
![image.png](1631777560691-21eeb1ae-c10f-4491-b80d-3a48132b2653.png)
，分别支持Biz-SIP开放平台接口访问和个性化的PersonController实现的REST接口：

- 通过Biz-SIP开放平台接口测试：
```java
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:app/personal" -X POST --data '{"methodName":"getCustomerAndAccountList","params":["001"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "c3cda0e1b95f449e986dc5ee41e48716",
  "parentTraceId": null,
  "timestamp": 1630760469452,
  "data": {
    "result": {
      "accountList": [
        {
          "accountId": "0001",
          "balance": 100,
          "customerId": "001"
        },
        {
          "accountId": "0002",
          "balance": 200,
          "customerId": "001"
        }
      ],
      "customer": {
        "sex": "1",
        "customerName": "张三",
        "customerId": "001",
        "age": 30
      }
    }
  }
}
```

- 通过个性化PersonController实现的REST接口，接口测试如下：
```java
$ curl http://localhost:9001/personal/getCustomerAndAccountList\?customerId=001|jq

{
  "customer": {
    "customerId": "001",
    "customerName": "张三",
    "age": 30,
    "sex": "1"
  },
  "accountList": [
    {
      "accountId": "0001",
      "customerId": "001",
      "balance": 100
    },
    {
      "accountId": "0002",
      "customerId": "001",
      "balance": 200
    }
  ]
}

```
## 六、项目实践：支付域服务的开发
### 1. Payment领域服务的封装
payment领域服务是对接第三方缴费平台的，第三方缴费平台的接口是XML报文格式。
payment领域服务是属于对接第三方的领域服务，前面提到的customer领域服务和account领域服务，主要是内部交易处理的领域服务。
这二类领域服务在开发时有比较大的不同：

- 对接第三方的领域服务，一般涉及到复杂的通讯接口对接和报文格式转换；
- 内部交易处理领域服务，应用层和领域层之间，建议采用约定的Interface接口类的调用约定（底层Biz-SIP平台自动转换成RESTful协议和BizMessage标准消息进行交互）。

在开发对接第三方的领域服务时，建议采用实现SinkBeanInterface接口的JavaBean或SpringBean：
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
​

Payment领域服务的封装，依次有以下步骤：
> 第1步

领域服务的实现：创建xbank-payment1-sink子模块，根据SinkBeanInterface接口，实现Payment1SinkService微服务，并封装成能独立运行的微服务应用：
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
> 第2步

在sink.yml文件中，把Payment1SinkService微服务作为Sink连接到Biz-SIP平台：
```yaml
- id: payment1-sink
  type: rest
  url: http://payment1-sink/sink
  converter:
    type: simple-xml
  connector:
    type: sink-bean
    class-name: com.xbank.sink.payment.service.Payment1SinkService
    spring-bean: true
```
### 2. Payment服务的快速发布
配置service.yml文件，把payment-sink直接作为sink-service暴露给Biz-SIP开放平台接口：
```yaml
- bizServiceId: sink/payment1
  type: sink-service
  sinkId: payment1-sink
```
接口测试如下：
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:sink/payment1" -X POST --data '{"message":"Hello world!"}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "af37bbbb495744d79f2f1811178436f5",
  "parentTraceId": null,
  "timestamp": 1631153112943,
  "data": {
    "message": "Hello world!"
  }
}

```
Payment1SinkApplication会打印出日志，收到了完成消息转换后的XML报文：
```
2021-09-09 10:05:13.265  INFO 10756 --- [nio-8003-exec-1] c.x.d.p.service.Payment1DomainService    : 传入消息:
====+ 01-02-03-04-05-06-07-08-09-10-11-12-13-14-15-16-17-18-19-20 + ====== ASCII  ====== +
0000: 3C 3F 78 6D 6C 20 76 65 72 73 69 6F 6E 3D 22 31 2E 30 22 20 | <?xml version="1.0"  |
0020: 65 6E 63 6F 64 69 6E 67 3D 22 55 54 46 2D 38 22 20 73 74 61 | encoding="UTF-8" sta |
0040: 6E 64 61 6C 6F 6E 65 3D 22 6E 6F 22 3F 3E 3C 72 6F 6F 74 3E | ndalone="no"?><root> |
0060: 3C 6D 65 73 73 61 67 65 3E 48 65 6C 6C 6F 20 77 6F 72 6C 64 | <message>Hello world |
0080: 21 3C 2F 6D 65 73 73 61 67 65 3E 3C 2F 72 6F 6F 74 3E       | !</message></root>.. |
```
### 3. payment领域服务在应用层和适配层的定制
通过在service.yml中配置sink-service聚合服务，能实现已经挂接到Sink的领域服务的快速发布。
但是，客户针对应用层和适配层，还是有个性化定制要求，这就涉及到应用层和适配层的定制。
例如我们要实现一个把消息发送给缴费平台的业务，输入的消息包括：交易码、传递的消息。
首先，需要在xbank-personal-app-client子模块中的PersonalAppInterface接口，增加一个消息发送接口send2payment1()，这个接口是共享给应用层和适配层的：
```java
public interface PersonalAppInterface {
    ...
    public BizMessage send2Payment1(Object message) throws BizException;
    ...
}
```
然后在xbank-app模块中基于此接口方法。在PersonalAppService类实现接口：
```java
public class PersonalAppService implements PersonalAppInterface {
	...
    private BizMessageInterface payment1DomainInterface = IntegratorClientFactory
            .getSinkClient(BizMessageInterface.class,"payment1-sink");
	...
    @Override
    public BizMessage<JSONObject> send2Payment1(Object message) throws BizException {
        JSONObject jsonObject = new JSONObject();
        jsonObject.set("message",message);
        return this.payment1DomainInterface.call(jsonObject);
    }
    ...
}
```
由于payment领域服务没有另外约定接口，是采用BizMessage标准消息接口，所以是采用BizMessageInterface接口来构建sink调用接口的，并统一通过call()方法来调用领域服务。
由于前面已经在service.yml中把PersonalAppService应用层服务类，配置成了“bean-service”的聚合服务，所以可以直接进行接口调用：
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:app/personal" -X POST --data '{"methodName":"send2Payment1","params":["Hello world!"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "19620fe5843d4d53a2bf1f599b516006",
  "parentTraceId": null,
  "timestamp": 1631153624870,
  "data": {
    "result": {
      "traceId": "19620fe5843d4d53a2bf1f599b516006",
      "code": 0,
      "data": {
        "message": "Hello world!"
      },
      "message": "success",
      "timestamp": 1631153624870
    }
  }
}
```
Payment1SinkApplication应用会收到消息：
```
2021-09-09 10:13:44.911  INFO 10756 --- [nio-8003-exec-2] c.x.d.p.service.Payment1DomainService    : 传入消息:
====+ 01-02-03-04-05-06-07-08-09-10-11-12-13-14-15-16-17-18-19-20 + ====== ASCII  ====== +
0000: 3C 3F 78 6D 6C 20 76 65 72 73 69 6F 6E 3D 22 31 2E 30 22 20 | <?xml version="1.0"  |
0020: 65 6E 63 6F 64 69 6E 67 3D 22 55 54 46 2D 38 22 20 73 74 61 | encoding="UTF-8" sta |
0040: 6E 64 61 6C 6F 6E 65 3D 22 6E 6F 22 3F 3E 3C 72 6F 6F 74 3E | ndalone="no"?><root> |
0060: 3C 6D 65 73 73 61 67 65 3E 48 65 6C 6C 6F 20 77 6F 72 6C 64 | <message>Hello world |
0080: 21 3C 2F 6D 65 73 73 61 67 65 3E 3C 2F 72 6F 6F 74 3E       | !</message></root>.. |
```
在适配层中，在原有的PersonalController类中，增加对“/send2Payment1"请求的实现：
```java
    @GetMapping(value ="/send2Payment1")
    public BizMessage<JSONObject> send2Payment1(String message) throws BizException {
        return this.personalAppInterface.send2Payment1(message);
    }
```
开发者可以在适配层的PersonalController类中，通过personalAppInterface，把消息发送给缴费平台：
```bash
$ curl http://localhost:9001/personal/send2Payment1\?message=hello|jq 

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "3c9d800b3c76480c9ae605d543709ea3",
  "parentTraceId": null,
  "timestamp": 1631154087879,
  "data": {
    "message": "hello"
  }
}
```
在适配层中，还可以通过配置sink-service类型的聚合服务，直接调用领域层的Sink服务。例如可以在原有的PersonalController类中，直接向应用层发起JSONObject类型的平台报文：
```java
@RestController
@RequestMapping("/personal")
public class PersonalController  {
	...
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
开发者可以在适配层的PersonalController类中，通过personalAppInterface，把消息发送给缴费平台：
```bash
$ curl http://localhost:9001/personal/send2Payment1\?message=hello|jq 

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "3c9d800b3c76480c9ae605d543709ea3",
  "parentTraceId": null,
  "timestamp": 1631154087879,
  "data": {
    "message": "hello"
  }
}
```
## 七、项目实践：应用服务对延迟服务的组装
### 1. SAF存储转发的实现
在领域层中，开发Payment2SinkService，根据tranMode交易模式，实现处理超时、处理失败和处理成功的各种异常情况：
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
注意，Payment2领域服务在封装时，涉及到多个接口调用，对每个接口采用命令执行器（Command Executor），这里是传入标准的JSONObject数据，所以所有的接口实现继承AbstractSinkBeanCmdExe类来实现：
![image.png](1631778769217-1eb52b38-45bd-4dff-b4d2-aa09cb2acd00.png)
在应用层中，在原有的PersonalAppInterface应用层接口、PersonalAppService服务类中，增加send2Payment2()、getCustomerAndSaf2Payment2()这2个方法接口和实现，通过personalAppDelayInterface延迟调用接口，实现对原有send2Payment2()方法的多次SAF调用：

```java
@Service
public class PersonalAppService implements PersonalAppInterface {
    ...
        
    private BizMessageInterface payment2SinkInterface = IntegratorClientFactory
            .getSinkClient(BizMessageInterface.class,"payment2-sink");
    private PersonalAppInterface personalAppDelayInterface = IntegratorClientFactory
            .getDelayBizServiceClient(PersonalAppInterface.class,"app/personal",
                    0,1000,2000,4000,8000,16000,32000);
	...
        
    @Override
    public BizMessage send2Payment2(String tranMode, String tranCode, Object message) throws BizException {
        JSONObject jsonObject = new JSONObject();
        jsonObject.set("tranCode",tranCode);
        jsonObject.set("tranMode",tranMode);
        jsonObject.set("message",message);
        return this.payment2SinkInterface.call(jsonObject);
    }

    @Override
    public Customer getCustomerAndSaf2Payment2(String tranMode,String customerId) throws BizException {
        Customer customer = this.customerSinkInterface.getCustomer(customerId);
        this.personalAppDelayInterface.send2Payment2(tranMode,"send-customer", customer);
        return customer;
    }
    ...
}
```
传入交易模式为”timeout“（收到交易后，永远返回超时），这将导致发送7次，最后交易状态为失败：
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:app/personal" -X POST --data '{"methodName":"getCustomerAndSaf2Payment2","params":["timeout","001"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "c1d2ffb7f0b44bf4b0cb7e13e5c0c0fa",
  "parentTraceId": null,
  "timestamp": 1631274896174,
  "data": {
    "result": {
      "sex": "1",
      "customerName": "张三",
      "customerId": "001",
      "age": 30
    }
  }
}

```
Payment2SinkApplication日志：
```
2021-09-10 19:54:56.650  INFO 14993 --- [nio-8004-exec-3] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"timeout","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:54:56.651  INFO 14993 --- [nio-8004-exec-3] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:54:57.753  INFO 14993 --- [nio-8004-exec-4] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"timeout","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:54:57.754  INFO 14993 --- [nio-8004-exec-4] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:54:59.920  INFO 14993 --- [nio-8004-exec-5] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"timeout","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:54:59.920  INFO 14993 --- [nio-8004-exec-5] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:55:04.015  INFO 14993 --- [nio-8004-exec-6] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"timeout","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:55:04.015  INFO 14993 --- [nio-8004-exec-6] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:55:12.041  INFO 14993 --- [nio-8004-exec-7] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"timeout","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:55:12.042  INFO 14993 --- [nio-8004-exec-7] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:55:28.080  INFO 14993 --- [nio-8004-exec-8] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"timeout","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:55:28.080  INFO 14993 --- [nio-8004-exec-8] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:56:00.228  INFO 14993 --- [nio-8004-exec-9] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"timeout","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:56:00.228  INFO 14993 --- [nio-8004-exec-9] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!

```


传入交易码为”3timeout-fail“（收到交易后，前3次返回超时，第4次返回失败），这将导致发送4次，最后交易状态为失败：
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:app/personal" -X POST --data '{"methodName":"getCustomerAndSaf2Payment2","params":["3timeout-fail","001"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "d3589b169d654a409c2f7d15f83f5113",
  "parentTraceId": null,
  "timestamp": 1631275051123,
  "data": {
    "result": {
      "sex": "1",
      "customerName": "张三",
      "customerId": "001",
      "age": 30
    }
  }
}

```
Payment2SinkApplication日志：
```
2021-09-10 19:57:31.379  INFO 14993 --- [nio-8004-exec-1] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"3timeout-fail","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:57:31.379  INFO 14993 --- [nio-8004-exec-1] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:57:32.647  INFO 14993 --- [nio-8004-exec-2] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"3timeout-fail","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:57:32.647  INFO 14993 --- [nio-8004-exec-2] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:57:34.672  INFO 14993 --- [nio-8004-exec-3] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"3timeout-fail","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:57:34.673  INFO 14993 --- [nio-8004-exec-3] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:57:38.733  INFO 14993 --- [nio-8004-exec-4] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"3timeout-fail","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:57:38.733  INFO 14993 --- [nio-8004-exec-4] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回交易处理失败!
```
传入交易码为”3timeout-success“（收到交易后，前3次返回超时，第4次成功返回原报文），这将导致发送4次，最后交易状态为成功：
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:app/personal" -X POST --data '{"methodName":"getCustomerAndSaf2Payment2","params":["3timeout-success","001"]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "75b76cf518164964bedf477ea488698c",
  "parentTraceId": null,
  "timestamp": 1631275117934,
  "data": {
    "result": {
      "sex": "1",
      "customerName": "张三",
      "customerId": "001",
      "age": 30
    }
  }
}
```
Payment2SinkApplication日志：
```
2021-09-10 19:58:38.146  INFO 14993 --- [nio-8004-exec-5] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"3timeout-success","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:58:38.146  INFO 14993 --- [nio-8004-exec-5] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:58:39.200  INFO 14993 --- [nio-8004-exec-6] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"3timeout-success","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:58:39.200  INFO 14993 --- [nio-8004-exec-6] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:58:41.409  INFO 14993 --- [nio-8004-exec-7] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"3timeout-success","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:58:41.409  INFO 14993 --- [nio-8004-exec-7] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回超时异常!
2021-09-10 19:58:45.539  INFO 14993 --- [nio-8004-exec-8] c.x.d.p.service.Payment2DomainService    : 传入消息:
{"tranMode":"3timeout-success","message":{"sex":"1","customerName":"张三","customerId":"001","age":30},"tranCode":"send-customer"}
2021-09-10 19:58:45.539  INFO 14993 --- [nio-8004-exec-8] c.x.d.p.service.Payment2DomainService    : 交易:send-customer,返回交易成功!
```
### 2. 向前补偿的实现
向前补偿机制是指当交易无应答时，系统会对原交易进行查询，如查询超时会重试多次。如原交易成功则继续后续交易，如原交易失败则对前续交易进行交易补偿，向前补偿一般是针对第三方接入方无法提供补偿交易时的解决方案。
在金融领域，这类交易补偿动作常被称为“冲正”。
在应用层中，在原有的PersonalAppInterface应用层接口、PersonalAppService服务类中，增加payoutForward()、payoutForwardCompensate()这2个方法接口和实现，通过personalAppDelayInterface延迟调用接口，实现对原有payoutForwardCompensate()方法的多次重复延迟调用：
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
    
    ......
        
	@Override
    public void payoutForward(String tranMode,String accountId, long amount) throws BizException {
        log.info("account出金:{},{}",accountId,amount);
        this.accountDomainInterface.payout(accountId,amount);
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

    @Override
    public void payoutForwardCompensate(JSONObject jsonObject)  throws BizException{
        jsonObject.set("tranCode","pay-query");
        BizMessage<JSONObject> bizMessage = null;
        try {
            log.info("payment查询缴费订单...");
            bizMessage = this.payment2SinkInterface.call(jsonObject);
        } catch (BizException e) {
            if (e.isTimeOutException()) {
                log.info("payment交易超时...");
                throw e;
            }
            else {
                log.info("payment查询缴费订单返回错误（表示对方订单没执行）...");
                String accountId = (String)jsonObject.get("accountId");
                long amount = (Integer) jsonObject.get("tranAmount");
                log.info("account出金补偿:{},{}",accountId,amount);
                this.accountDomainInterface.payoutCompensation(accountId,amount);
                return;
            }
        }
        log.info("payment查询缴费订单成功（表示对方订单已执行）");
        log.info("payout成功!");
    }
```
传入交易模式为”3timeout-success“（收到订单交易后，前3次返回超时，第4次成功返回模拟订单已执行），这将导致发送4次，最后交易状态为成功：
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:app/personal" -X POST --data '{"methodName":"payoutForward","params":["3timeout-success","0001",1]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "e0127a25e2514da682ad77042917087a",
  "parentTraceId": null,
  "timestamp": 1631275840559,
  "data": {}
}
```
XbankAppApplication日志：
```
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:10:40.560 INFO 16503 [11034350FE38437BB459F1E8DC6CCE17] [http-nio-8888-exec-5] c.xbank.app.service.PersonalAppService   account出金:0001,1
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:10:40.822 INFO 16503 [11034350FE38437BB459F1E8DC6CCE17] [http-nio-8888-exec-5] c.xbank.app.service.PersonalAppService   payment缴费...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:10:40.831 INFO 16503 [11034350FE38437BB459F1E8DC6CCE17] [http-nio-8888-exec-5] c.xbank.app.service.PersonalAppService   payment交易超时,开始payout补偿...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:10:42.086 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-4] c.xbank.app.service.PersonalAppService   payment查询缴费订单...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:10:42.095 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-4] c.xbank.app.service.PersonalAppService   payment交易超时...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:10:43.268 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-3] c.xbank.app.service.PersonalAppService   payment查询缴费订单...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:10:43.277 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-3] c.xbank.app.service.PersonalAppService   payment交易超时...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:10:45.296 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-2] c.xbank.app.service.PersonalAppService   payment查询缴费订单...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:10:45.306 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-2] c.xbank.app.service.PersonalAppService   payment查询缴费订单成功（表示对方订单已执行）
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:10:45.306 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-2] c.xbank.app.service.PersonalAppService   payout成功!
```
传入交易模式为”3timeout-success“（收到订单交易后，前3次返回超时，第4次成功返回模拟订单已执行），这将导致发送4次，最后交易状态为成功：
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:app/personal" -X POST --data '{"methodName":"payoutForward","params":["3timeout-fail","0001",1]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "a04163bf05484fa28071650a085d4124",
  "parentTraceId": null,
  "timestamp": 1631275997342,
  "data": {}
}

```
XbankAppApplication日志：
```
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:13:17.342 INFO 16503 [5E5951618A27430684C5552E1F782D8B] [http-nio-8888-exec-7] c.xbank.app.service.PersonalAppService   account出金:0001,1
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:13:17.568 INFO 16503 [5E5951618A27430684C5552E1F782D8B] [http-nio-8888-exec-7] c.xbank.app.service.PersonalAppService   payment缴费...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:13:17.579 INFO 16503 [5E5951618A27430684C5552E1F782D8B] [http-nio-8888-exec-7] c.xbank.app.service.PersonalAppService   payment交易超时,开始payout补偿...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:13:17.775 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-5] c.xbank.app.service.PersonalAppService   payment查询缴费订单...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:13:17.788 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-5] c.xbank.app.service.PersonalAppService   payment交易超时...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:13:18.827 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-1] c.xbank.app.service.PersonalAppService   payment查询缴费订单...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:13:18.836 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-1] c.xbank.app.service.PersonalAppService   payment交易超时...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:13:20.890 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-4] c.xbank.app.service.PersonalAppService   payment查询缴费订单...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:13:20.900 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-4] c.xbank.app.service.PersonalAppService   payment查询缴费订单返回错误（表示对方订单没执行）...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:13:20.901 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-4] c.xbank.app.service.PersonalAppService   account出金补偿:0001,1
```
### 3. 向后补偿的实现
向后补偿机制是指当交易无应答时，系统会对原交易发起多次交易补偿重试，如补偿交易成功则对前续交易进行补偿。
在应用层中，在原有的PersonalAppInterface应用层接口、PersonalAppService服务类中，增加payoutBackward()、payoutBackwardCompensate()这2个方法接口和实现，通过personalAppDelayInterface延迟调用接口，实现对原有payoutForwardCompensate()方法的多次重复延迟调用：
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
    
    ......
        
	@Override
    public void payoutBackward(String tranMode, String accountId, long amount) throws BizException {
        log.info("account出金:{},{}",accountId,amount);
        this.accountDomainInterface.payout(accountId,amount);
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
                log.info("payment交易超时,开始payout冲正...");
                this.personalAppDelayInterface.payoutBackwardCompensate(jsonObject);
                return;
            }
            else {
                log.info("payment缴费交易返回错误");
                log.info("account出金补偿:{},{}",accountId,amount);
                this.accountDomainInterface.payoutCompensation(accountId,amount);
                log.info("payout交易补偿成功！");
                return;
            }
        }
        log.info("payment缴费成功");
        log.info("payout成功!");
    }

    @Override
    public void payoutBackwardCompensate(JSONObject jsonObject) throws BizException {
        jsonObject.set("tranCode","pay-reversal");
        BizMessage<JSONObject> bizMessage;
        try {
            log.info("payment缴费补偿...");
            bizMessage = this.payment2SinkInterface.call(jsonObject);
            log.info("payment缴费补偿成功");
        } catch (BizException e) {
            if (e.isTimeOutException()) {
                log.info("payment缴费补偿交易超时...");
                throw e;
            }
            log.info("payment缴费补偿交易失败,需要人工干预调整!");
            return;
        }
        String accountId = (String)jsonObject.get("accountId");
        long amount = (Integer)jsonObject.get("tranAmount");
        log.info("account出金补偿:{},{}",accountId,amount);
        this.accountDomainInterface.payoutCompensation(accountId,amount);
        log.info("payout补偿成功！");
    }
```
传入交易模式为”3timeout-success“（收到订单交易后，前3次返回补偿交易超时，第4次成功返回补偿交易执行成功），这将导致发送4次，最后交易状态为成功：
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:app/personal" -X POST --data '{"methodName":"payoutBackward","params":["3timeout-success","0001",1]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "26c58fe44b3f4ceba462762494fa5981",
  "parentTraceId": null,
  "timestamp": 1631276405513,
  "data": {}
}
```
XbankAppApplication日志：
```java
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:05.513 INFO 16503 [84936F44451449C5A92183EF93C3690C] [http-nio-8888-exec-2] c.xbank.app.service.PersonalAppService   account出金:0001,1
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:05.952 INFO 16503 [84936F44451449C5A92183EF93C3690C] [http-nio-8888-exec-2] c.xbank.app.service.PersonalAppService   payment缴费...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:05.961 INFO 16503 [84936F44451449C5A92183EF93C3690C] [http-nio-8888-exec-2] c.xbank.app.service.PersonalAppService   payment交易超时,开始payout冲正...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:06.396 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-1] c.xbank.app.service.PersonalAppService   payment缴费补偿...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:06.405 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-1] c.xbank.app.service.PersonalAppService   payment缴费补偿交易超时...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:07.468 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-4] c.xbank.app.service.PersonalAppService   payment缴费补偿...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:07.475 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-4] c.xbank.app.service.PersonalAppService   payment缴费补偿交易超时...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:09.560 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-3] c.xbank.app.service.PersonalAppService   payment缴费补偿...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:09.570 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-3] c.xbank.app.service.PersonalAppService   payment缴费补偿成功
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:09.571 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-3] c.xbank.app.service.PersonalAppService   account出金补偿:0001,1
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:20:09.726 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-3] c.xbank.app.service.PersonalAppService   payout补偿成功！
```
传入交易模式为”3timeout-fail“（收到订单交易后，前3次返回超时，第4次成功返回模拟补偿交易失败），这将导致发送4次，最后交易状态为失败：
```bash
$ curl -H "Content-Type:application/json" -H "Biz-Service-Id:app/personal" -X POST --data '{"methodName":"payoutBackward","params":["3timeout-fail","0001",1]}' http://localhost:8888/api|jq

{
  "code": 0,
  "message": "success",
  "extMessage": null,
  "traceId": "40aed8f06bba436ba67ff93e769d5a33",
  "parentTraceId": null,
  "timestamp": 1631276555599,
  "data": {}
}
```
XbankAppApplication日志：
```
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:22:35.599 INFO 16503 [B54A535614A64492A7A00D987F12B529] [http-nio-8888-exec-4] c.xbank.app.service.PersonalAppService   account出金:0001,1
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:22:35.760 INFO 16503 [B54A535614A64492A7A00D987F12B529] [http-nio-8888-exec-4] c.xbank.app.service.PersonalAppService   payment缴费...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:22:35.767 INFO 16503 [B54A535614A64492A7A00D987F12B529] [http-nio-8888-exec-4] c.xbank.app.service.PersonalAppService   payment交易超时,开始payout冲正...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:22:35.979 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-2] c.xbank.app.service.PersonalAppService   payment缴费补偿...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:22:35.993 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-2] c.xbank.app.service.PersonalAppService   payment缴费补偿交易超时...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:22:37.073 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-5] c.xbank.app.service.PersonalAppService   payment缴费补偿...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:22:37.083 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-5] c.xbank.app.service.PersonalAppService   payment缴费补偿交易超时...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:22:39.111 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-1] c.xbank.app.service.PersonalAppService   payment缴费补偿...
[bizsip-integrator:192.169.1.101:8888] 2021-09-10 20:22:39.121 INFO 16503 [] [org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-1] c.xbank.app.service.PersonalAppService   payment缴费补偿交易失败,需要人工干预调整!
```
## 八、总结
### 1. Sink服务（领域层服务）

- sink-bean类sink服务
   - sink接口为BizMessage<JSONObject>类型；
   - 基于<SinkBeanInterface>或<JSONObjectSinkBeanInterface>接口开发服务类，实现process()方法，2个接口区别在于出入参数的类型，前1个是byte[]类型，后1个是JSONObject类型；
   - 在sink.yml中配置”sink-bean“类型的connector，服务类支持JavaBean和SpringBean二种挂接方式；
   - 在sink.yml中支持多种convert，传入process()方法的参数为经过convert转换后的消息。
- bean类sink服务
   - sink接口为BizMessage<JSONObject>类型，JSONObject消息体中包括：className（可选）、methodName（可选）、params（必选），params为JSONArray类型；
   - 可自由开发服务类；
   - 在sink.yml中配置”bean“类型的connector，服务类支持JavaBean和SpringBean二种挂接方式；
   - 在sink.yml中convert只支持simple-json类型（后期可能会取消对于bean类sink服务的convert配置）。
### 2. Biz-Service聚合服务（应用层服务）
integrator接口为带”Biz-Service-Id“header头、以POST方式提交JSON数据，返回为BizMessage<JSONObject>类型的JSON串。
在integrator-bean-service、bean-service类聚合服务开发服务类中，可以用IntegratorClientFactory.getSinkClient()和IntegratorClientFactory.getDelayBizServiceClient()，分别获取sink接口访问和延迟Biz-Service聚合服务接口访问的client API接口。
对sink-bean类sink服务，sink接口为BizMessage<JSONObject>类型，应采用<BizMessageInterface>接口，统一调用call()方法。

- integrator-bean-service类聚合服务
   - 请求提交为JSON数据；
   - 基于<IntegratorBeanInterface>接口开发服务类，实现doBizService()方法；
   - 在service.yml中配置”integrator-bean-service“类型的聚合服务，服务类目前只支持JavaBean挂接方式。
- bean-service类聚合服务
   - 请求提交为JSON数据，JSON数据中包括：methodName（必选）、params（必选），params为JSONArray类型；
   - 可自由开发服务类；
   - 在service.yml中配置”bean-service“类型的聚合服务，服务类目前只支持SpringBean挂接方式。
- sink-service类聚合服务
   - 请求提交为JSON数据，JSON数据根据后端sink服务接口来定；
   - 无需开发代码；
   - 在service.yml中配置”sink-service“类型的聚合服务，并配置关联到后端的sink端口。
- script类聚合服务
   - 请求提交为JSON数据即可，在脚本中调用sink前，应按要求转换成sink约定的JSON数据格式；
   - 无需开发Java代码，但需配置聚合服务脚本；
   - 在config/service目录中配置脚本文件，无需在service.yml中配置。
### 3. 适配层服务

- 采用Biz-SIP自带OpenAPI接口
   - Integrator接口为Biz-SIP平台的OpenAPI接口，可以按需对外开放访问。
- 定制适配层接入模块
   - 引入Souce类，可以根据消息格式转换的定制，自动进行消息格式转换和适配；
   - 通过SourceClientFactory.getBizServiceClient()，获取Biz-Service聚合服务访问接口（具体请参见[链接](#lazZY)）。
## 附录1：Biz-SIP运行机制图
![image.png](1631863592446-fa064970-8fc9-4d81-b1dc-8b601c639846.png)
## 附录2：DDD各层功能定位与粒度划分
| 层级 | 功能定位 | 粒度划分 |
| --- | --- | --- |
| 适配层 | 渠道接入相关的加解密、验签；<br>渠道接入通讯连接的对接；<br>渠道消息格式的转换；<br>OpenAPI的接入； | 按接入渠道进行划分；<br>渠道入口和出口，可能会拆分，一般渠道入口在适配层，出口由领域层对接； |
| 应用层 | 统一应用层消息校验；<br>领域层服务的组装和编排； | 按业务、按系统进行拆分，涉及到1个以上领域服务； |
| 领域层 | 交易处理；<br>第三方对接，包括安全（加解密、验签）、通讯、消息格式的转换； | 交易系统，按业务划分；<br>对接第三方，按渠道；<br>按数据库划分； |
| 基础设施层 | 数据库DAO；<br>HSM安全 |  |

