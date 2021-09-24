---
title: 接口规范
date: '2021-09-24'
type: book
weight: 50
---

介绍Biz-SIP中间件各模块间的接口类型和通讯方式。

<!--more-->

## 一、Biz-SIP内部标准消息规范
Biz-SIP中间件的内部消息规范如下：

| 参数 | 类型 | 是否必填 | 描述 |
| --- | --- | --- | --- |
| code | int | Y | 返回码，0为成功，非0为失败 |
| message | String | N | 返回消息 |
| extMessage | String | N | 返回扩展消息 |
| traceId | char(32) | Y | 由Biz-SIP统一生成的唯一跟踪ID，每个聚合服务中traceId相同 |
| parentTraceId | char(32) | N | 父交易服务的traceId，父交易服务一般调用SAF服务，会产生子交易服务 |
| timestamp | long | Y | 由Biz-SIP统一生成的时间戳，为聚合服务的最初发起时间，为1970年1月1日零点整至发起时间的毫秒数 |
| data | String（JSON格式） | N | 传送的数据，一般为JSON格式 |

## 二、渠道层Open API接口规范
Open API接口网关，是Biz-SIP中间件对外提供的标准接口接入，规范如下：

| URL | http://{地址}:{端口}/api |
| --- | --- |
| HTTP请求头 |  |
| - Content-Type | application/json |
 | - Biz-Service-Id | 调用的聚合服务ID |
 | 请求包 | JSON报文 |
 | 响应包 | Biz-SIP内部标准消息（参见“Biz-SIP内部标准消息规范”） |

## 三、应用层App Integrator接口规范
服务整合器（Integrator）接口，对外是以Spring Cloud微服务的形式提供的，内部调用是采用这种接口规范，同时接入非标接口的客户端适配器也是采用这个接口进行接入的，规范如下：

| URL | http://sip-integrator/api |
| --- | --- |
| HTTP请求头 |  |
| - Content-Type | application/json |
  | - Biz-Service-Id| 调用的聚合服务ID |
  | 请求包 | JSON报文 |
  | 响应包 | Biz-SIP内部标准消息（参见“Biz-SIP内部标准消息规范”） |

## 四、领域层Sink接口规范
Sink接口，是Integrator调用所有Sink模块的接口，规范如下：

| Content-Type | application/json |
| --- | --- |
| 请求包 | Biz-SIP内部标准消息（参见“Biz-SIP内部标准消息规范”） |
| 响应包 | Biz-SIP内部标准消息（参见“Biz-SIP内部标准消息规范”） |

