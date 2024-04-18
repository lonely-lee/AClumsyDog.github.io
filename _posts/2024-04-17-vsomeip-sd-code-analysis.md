---
title: vsomeip-sd模块源码分析
date: 2024-04-17 14:57 +0800
last_modified_at: 2024-04-18 14:31 +0800
author: AClumsyDog
categories: ["源码分析", "vsomeip"]
tags: ["c++", "someip", "autosar"]
pin: true
math: true
mermaid: true
img_path: /assets/img/vsomeip-sd-code-analysis/
---

## 服务发现UML类图

服务发现模块类之间的关系较为复杂，为了方便理解。这里先对各个主要类进行简单的说明。

**注意**：类的详细说明见链接。

- **runtime和runtime_impl**：

  service discovery类的构造工厂类。可以创建service_discovery_impl类的实例。runtime是纯虚类，runtime_impl继承runtime。

- **service_discovery_host**：

  纯虚类，Routing Manager相关类（没有包括在此UML类图中）继承于此类。service_discovery_impl类中包括指向此类的指针，所以service_discovery_impl可以通过指向service_discovery_host的指针间接使用Routing Manager相关类（例如：发送消息）。

- **service_discovery和service_discovery_impl**：

  service_discovery是纯虚类，service_discovery_impl继承service_discovery类实现所有的纯虚函数。服务发现相关的处理逻辑在此类中实现。

- **request**：

  保存请求服务相关的信息。service_discovery_impl中保存有request的容器，可以处理相关服务发现的逻辑。

- **subscription**：

  pass

- **remote_subscription_ack**：

  pass

- **message_element_impl**：

  所有message相关类的基类，message_elemen_impl中有指向message_impl的指针。其他类（entry_impl相关类和option_impl相关类）可以通过调用`get_owning_message()`获得指向message_impl类指针。

- **message_impl**：

  组合所有message相关的类，`entries_`（`std::vector<std::shared_ptr<entry_impl>>`）和`options_`（`std::vector<std::shared_ptr<option_impl>>`）保存了所有的entry_impl相关类和option_impl相关类。

- **entry_impl**：

  entry相关类的基类。

- **option_impl**：

  option相关类的基类。

```mermaid
classDiagram
    class message_impl {
        -entries_t entries_
        -options_t options_
        +get_entries() const entries_t&
        +get_options() const options_t&
        +add_entry_data(const std::shared_ptr~entry_impl~&, const std::vector~std::shared_ptr~option_impl~~&, const std::shared_ptr~entry_impl~&) bool
        +find_option(const std::shared_ptr~option_impl~&) std::shared_ptr~option_impl~
        +get_option_index(const std::shared_ptr~option_impl~&) int16_t
        +get_option(int16_t) std::shared_ptr~option_impl~
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
        -deserialize_entry(vsomeip_v3::deserializer *_from) entry_impl*
        -deserialize_option(vsomeip_v3::deserializer *_from) option_impl*
    }

    message_impl o-- option_impl : Aggregation
    message_impl o-- entry_impl : Aggregation

    class message_element_impl {
        -message_impl* owner_
        +message_element_impl()
        +get_owning_message() message_impl*
        +set_owning_message(message_impl* _owner)
    }
    message_element_impl --> message_impl : Association

    class option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    message_element_impl <|-- option_impl : Inheritance

    class configuration_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    option_impl <|-- configuration_option_impl : Inheritance

    class selective_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    option_impl <|-- selective_option_impl : Inheritance

    class load_balancing_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    option_impl <|-- load_balancing_option_impl : Inheritance

    class protection_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    option_impl <|-- protection_option_impl : Inheritance

    class ip_option_impl {
    	<<Abstract>>
        +serialize(vsomeip_v3::serializer*) bool*
        +deserialize(vsomeip_v3::deserializer*) bool*
    }
    option_impl <|-- ip_option_impl : Inheritance

    class ipv4_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    ip_option_impl <|-- ipv4_option_impl : Inheritance

    class ipv6_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    ip_option_impl <|-- ipv6_option_impl : Inheritance

    class entry_impl {
        +assign_option(const std::shared_ptr~option_impl~&)
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    message_element_impl <|-- entry_impl : Inheritance
    entry_impl ..> option_impl : Dependency

    class eventgroupentry_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
        +matches(const eventgroupentry_impl&, const message_impl::options_t&) bool
        +get_selective_option() std::shared_ptr~selective_option_impl~
    }
    entry_impl <|-- eventgroupentry_impl : Inheritance
	eventgroupentry_impl ..> selective_option_impl : Dependency
	eventgroupentry_impl ..> option_impl : Dependency

    class serviceentry_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    entry_impl <|-- serviceentry_impl : Inheritance

	class deserializer {
		+deserialize_sd_message message_impl*
	}
	deserializer ..> message_impl : Dependency

    class remote_subscription_ack {
        -std::vector~std::shared_ptr~message_impl~~ messages_
        +get_messages() std::vector~std::shared_ptr~message_impl~~
        +get_current_message() std::shared_ptr~message_impl~
        +add_message() std::shared_ptr~message_impl~
    }
    remote_subscription_ack o-- message_impl : Aggregation

    class request

    class runtime {
    	<<Abstract>>
    	+create_service_discovery(service_discovery_host*, std::shared_ptr~configuration~) std::shared_ptr~service_discovery~*
    }

    class runtime_impl {
    	+create_service_discovery(service_discovery_host*, std::shared_ptr~configuration~) std::shared_ptr~service_discovery~*
    }
    runtime <|-- runtime_impl : Inheritance
    runtime_impl ..> service_discovery_host : Dependency
    runtime_impl ..> service_discovery : Dependency

    class subscription

	class service_discovery {
		<<Abstract>>
	}

	class service_discovery_host {
		<<Abstract>>
	}

	class service_discovery_impl {
		-service_discovery_host *host_
		-std::shared_ptr~deserializer~ deserializer_
		-requests_t requested_
		-std::map~service_t, std::map~instance_t, std::map~eventgroup_t, std::shared_ptr~subscription~~~~ subscribed_
		-std::map~std::shared_ptr~remote_subscription~, std::shared_ptr~remote_subscription_ack~~ pending_remote_subscriptions_
		-process_serviceentry(std::shared_ptr~serviceentry_impl~&, const std::vector~std::shared_ptr~option_impl~~&, bool, std::vector~std::shared_ptr~message_impl~~&, bool, const sd_acceptance_state_t&)
		-process_offerservice_serviceentry(service_t, instance_t, major_version_t, minor_version_t, ttl_t, const boost::asio::ip::address&, uint16_t, const boost::asio::ip::address&, uint16_t, std::vector~std::shared_ptr~message_impl~~&, bool, const sd_acceptance_state_t&)
		-send(const std::vector~std::shared_ptr~message_impl~~&) bool
		-serialize_and_send(const std::vector~std::shared_ptr~message_impl~~&, const boost::asio::ip::address&)
		-create_subscription(major_version_t, ttl_t, const std::shared_ptr~endpoint~&, const std::shared_ptr~endpoint~&, const std::shared_ptr~eventgroupinfo~&) std::shared_ptr~subscription~
		-send_subscription(const std::shared_ptr~subscription~&, const service_t, const instance_t, const eventgroup_t, const client_t)
		-insert_find_entries(std::vector~std::shared_ptr~message_impl~~&, const requests_t &)
		-find_request(service_t, instance_t) std::shared_ptr~request~
		-insert_subscription_ack(const std::shared_ptr~remote_subscription_ack~&, const std::shared_ptr~eventgroupinfo~&, ttl_t, const std::shared_ptr~endpoint_definition~&, const std::set~client_t~&)
		-send_subscription_ack(const std::shared_ptr~remote_subscription_ack~&)
	}
	service_discovery <|-- service_discovery_impl : Inheritance
	service_discovery_impl --> service_discovery_host : Association
	service_discovery_impl o-- deserializer : Aggregation
	service_discovery_impl ..> message_impl : Dependency
	service_discovery_impl o-- subscription : Aggregation
	service_discovery_impl o-- request : Aggregation
	service_discovery_impl o-- remote_subscription_ack : Aggregation
```

