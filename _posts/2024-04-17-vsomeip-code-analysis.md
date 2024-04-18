---
title: vsomeip源码分析
date: 2024-04-17 14:45 +0800
last_modified_at: 2024-04-17 17:56 +0800
author: AClumsyDog
categories: ["源码分析", "vsomeip"]
tags: ["c++", "someip", "autosar"]
pin: true
math: true
mermaid: true
img_path: /assets/img/vsomeip-code-analysis/
---

## 什么是SOME/IP？

**Scalable service-Oriented MiddlewarE over IP**，是指基于 IP 的可扩展的**面向服务**的中间件。

SOME/IP 协议采用 C/S（Client/Server）的通信架构，，我们把**请求服务**的 ECU 看成是 **Client**，而**提供服务**的 ECU 就是 **Server**。根据服务接口类型，使用远程服务调用（Remote Procedure Call）机制，通过数据序列化和反序列化（Serialization/Deserialization）使得数据得以在网络中传输。通过可用服务发现 SD（Service Discovery）机制来实现服务的动态配置。

由上可见，SOME/IP 主要可以提供以下功能：

1. **数据序列化和反序列化（Serialization/Deserialization）**：服务通信数据与二进制数据流之间的双向转换；
2. **可用服务发现 SD（Service Discovery）**：管理服务状态，发现和提供服务，动态配置 SOME/IP 报文发送；
3. **服务发布与订阅（Publish/Subscribe）**：管理服务的发布与订阅关系；
4. **远程服务调用（Remote Procedure Call）**：实现控制器（Client）使用网络内其他控制器（Server）提供的服务。

## SOME/IP-SD （服务发现协议）

### 什么是SOME/IP-SD？

SOME/IP-SD 协议是 SOME/IP 协议的一种，其中 SD 是服务发现 **Service Discovery** 的简称。基于服务的通信需要由 Server 和 Client 共同完成，因此在服务创建并且可用之后，Server 和 Client 需要通过 SOME/IP-SD **动态创建**两者之间的连接。Client 可以远程调用Server 提供的服务，或者订阅 Server 发布的内容，**但是**在 Client 调用服务或者订阅内容之前，**需要**知道 Server 提供哪些服务，这个过程就是通过服务发现来实现的。

SOME/IP-SD 是服务的**信息清单**及**管理机制**，主要实现**服务寻址**及**事件订阅**两种功能。对服务进行寻址时，服务提供者（Server 端）通过服务发现（SD）通知其他 ECU（Client 端）有哪些服务可用，并间接地通知该服务的**地址信息**（Server 端 **IP 地址、端口号、协议**），服务消费者（Client 端）了解到某服务的状态后，能够调用该服务的相关内容。

### SOME/IP-SD 的主要功能

SOME/IP-SD 有**两个**主要功能：**一是**应用程序之间传达自己的服务或获取对方的服务是否可用。**二是**向其他应用程序订阅服务，也就是 **Client** 端通过 **SOME/IP-SD** 对服务进行订阅，然后 **Server** 端用 **SOME/IP** 里的 Notification 类型消息发布订阅内容。

SOME/IP-SD 报文主要有以下几类：

1. **OfferService**：Server 的服务 Ready 并满足发布条件后，主动发出 OfferService报文，告知组播内其他节点该服务已经启动，可以创建连接。
2. **FindService**：当 Client 在网络中未收到相关的 **OfferService** 报文，或者暂时未收到，而 Client 又需要访问该服务，那么 Client 就可以发送 **FindService** 报文主动寻找服务，如果 Server 服务 Ready，会回复 **OfferService** 报文。
3. **StopOffserService**：当 Server 发现服务不可用，不满足服务发布条件时，会主动发送 **StopOffserService** 报文，告知组播内其他节点，该服务已不可用，停止服务支持。
4. **Subscribe**：事件组的交互采用 “订阅 - 发布” 机制，当 Client 收到OfferService 报文之后，通过发送 Subscribe 报文主动跟 Server 订阅相关事件组。
5. **SubscribeACK/SubscribeNACK**：当 Server 收到 Client 的订阅报文之后，**需要**先行判断该 Client 是否符合可订阅的条件，**如果**该 Client 满足事件组订阅条件，则发送**SubscribeACK**，告知 Client 订阅成功。此后当事件组内的事件准备就绪之后，Server 会以约定好的形式发送相关事件给成功订阅的 Client。而如果该 Client 不符合事件组订阅条件，Server 会直接回复 SubscribeNACK，告知订阅不成功。
6. **StopSubscribe**：当 Client 订阅某个事件组之后，发现后续并**不再需要**该事件组的数据了，可发送 **StopSubscribe** 报文向 Server 取消订阅相应事件。

SOME/IP 可用在 TCP/UDP 协议上，但是 SOME/IP-SD **只能用在 UDP 上**，因为SOME/IP-SD 只是为了发现服务，并不需要 TCP 的可靠性连接等特性。

### SOME/IP-SD 协议报文格式

SOME/IP-SD 协议的报文格式如下：

![SOMEIP-SD-Header-Format](SOMEIP-SD-Header-Format.png)

从上表可知，你会发现字段 Message ID、Protocol Version、Interface Version、Message Type、Return Code 的值都是固定的，SOME/IP-SD 的 Service ID 是 **0xFFFF**，Method ID 是 **0x8100** 。另外，SOME/IP-SD 的 Client ID 必须是 **0x0000**，因为每个设备上只有一个 SD Instance；Session ID 则是从 1 开始，逐次累加，直到翻转后再次从 1 开始（Session ID 不能设为 0）。单播和组播，**每种通信关系单独维护一个 Session ID，每组 发送-接收 关系组也单独维护一个 Session ID。**

#### Flags 的各个 Field 描述

其中这个 **8-bit Flags** 的各个位段格式如下：

|                     | Bit 7 （Reboot Flag）        | Bit 6（Unicast Flag） | Bit 5（Explicit initialdata control flag） | Bits[4:0] |
| ------------------- | ---------------------------- | --------------------- | ------------------------------------------ | --------- |
| Value & Description | 0 -> Session ID wraps around | 0 -> No Support       | 1（Shall always set to 1？？？）           | 0         |
|                     | 1 -> Reboot                  | 1 -> Support          |                                            |           |

#### Entry Type and Format

存在 2 种类型的 Entry：**Service** 和 **Eventgroup**，这两种 Entry 的长度都是 **16byte**。

两种类型 Entry 的**格式**如下：

- **Service**
  ![SOMEIP-SD-Service-Entry-Type](SOMEIP-SD-Service-Entry-Type.png)
- **Eventgroup**
  ![SOMEIP-SD-Eventgroup-Entry-Type](SOMEIP-SD-Eventgroup-Entry-Type.png)

上表中的 **X** 是一个 1-bit 的标志 **Initial data Requested Flag**，如果需要 Server 发送初始数据，就需要将其置 1 。

两种类型 Entry **各个位段的赋值**如下：

|               | Service                          | Eventgroup                                 |
| ------------- | -------------------------------- | ------------------------------------------ |
| **Type**      | **0x00**：`FindService`          | **0x06**：`（Stop）SubscribeEventgroup`    |
|               | **0x01**：`（Stop）OfferService` | **0x07**：`SubscribeEventgroupAck（NACK）` |
| Service ID    | X                                | X*                                         |
| Instance ID   | X                                | X*                                         |
| Major Version | X                                | X*                                         |
| TTL           | X**                              | X**                                        |
| Minor Version | X                                | -                                          |
| Counter       | -                                | X                                          |
| Eventgroup ID | -                                | X*                                         |

- *：All "F" are forbiden.
- **：TTL = 0 for Stop and NACK.
- Different Major Version are not compatible
- Different Minor Version are compatible
- Counter：Different Subscriptions

[sd模块源码分析](/posts/vsomeip-sd-code-analysis)