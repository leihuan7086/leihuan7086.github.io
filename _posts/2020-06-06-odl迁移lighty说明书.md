---
title: odl-pce项目从karaf架构迁移到lighty.io架构
date: 2020-06-06
category: OpenDaylight
tags: Lighty
description: 
---



## 版本选择

1）项目基线[transportpce-stable-sodium](https://github.com/opendaylight/transportpce/tree/stable/sodium)版本：基于odlparent-5.0.1的transportpce-0.4.0

2）lighty-core版本：[lighty-core-11.0.1-release](https://github.com/PantheonTechnologies/lighty-core/releases/tag/11.0.1)（依赖odlparent-5.0.1）

3）lighty版本：基于[transportpce-5b93dd84b141e6afbd5899d6b7cdf6cc2afba82f](https://github.com/opendaylight/transportpce/tree/5b93dd84b141e6afbd5899d6b7cdf6cc2afba82f)版本的lighty-11.0.0（依赖odlparent-5.0.1、transportpce-0.4.0），手动修改为lighty-11.0.1（编译后版本号与lighty-core一致）

*说明：lighty-core为官方版本，源码尽量不修改直接编译。lighty模块为项目迁移开发模块，会调用transportpce模块的代码，可能会有小部分编译问题，需要手动处理。*

## 前置条件

jdk版本>=1.8

[maven版本>=3.5.0](https://maven.apache.org/download.cgi)

[odl镜像配置文件[settings.xml]](https://github.com/opendaylight/odlparent/blob/master/settings.xml)

## lighty模块开发

lighty.io不支持osgi/karaf，odl迁移到lighty时需要删除osgi/karaf依赖（transportpce天然没有这些依赖），buluprint文件bean注册机制迁移到lighty后需要手动编码实例化注册，具体步骤如下：

### 1）添加依赖pom

lighty模块的pom.xml中添加依赖：transportpce、lighty-core

```xml
<!-- 部分pom依赖示例 -->
<dependency>
	<groupId>org.opendaylight.transportpce</groupId>
	<artifactId>transportpce-pce</artifactId>
	<version>${transportpce.version}</version>
</dependency>
<dependency>
	<groupId>io.lighty.modules</groupId>
    <artifactId>lighty-restconf-nb-community</artifactId>
    <version>${lighty-core.version}</version>
</dependency>
```

### 2）加载模型yang-models

TPCEUtils类中添加transportpce编译后的yang-models目录

```java
// 部分代码示例
public static final Set<YangModuleInfo> TPCE_MODELS = ImmutableSet.of(          org.opendaylight.yang.gen.v1.http.org.openroadm.alarm.rev161014.$YangModuleInfoImpl.getInstance(),         org.opendaylight.yang.gen.v1.http.org.openroadm.alarm.rev181019.$YangModuleInfoImpl.getInstance()
);
```

### 3）初始化注册beans

TransportPCEImpl类中TransportPCEImpl(LightyServices)构造函数注册beans

```java
// 部分代码示例
public TransportPCEImpl(LightyServices lightyServices) {
    LOG.info("Creating PCE beans ...");
    pathComputationService = new PathComputationServiceImpl(networkTransaction, lightyServices.getBindingNotificationPublishService());
    // 实例化PceProvider
    pceProvider = new PceProvider(lightyServices.getRpcProviderService(), pathComputationService);
}

@Override
protected boolean initProcedure() {
    // 调用pce模块的PceProvider.init()注册
    pceProvider.init();
    return true;
}
```

### 4）lighty-core插件开发

官方介绍lighty.io支持的odl的插件功能，相关插件功能参考lighty-core各子模块下README.md介绍

```xml
What OpenDaylight components are available in lighty.io

The lighty.io core contains base ODL services and components:

MD-SAL, Controller, YANG Tools
Northbound plugins: RESTCONF, NETCONF
Southbound plugins: SNMP, NETCONF, OpenFlow 
Apps: TransportPCE, ONAP’s SDN-C
```

## 编译&运行

### 1）编译transportpce

```sh
cd transportpce-stable-sodium
mvn clean install -DskipTests
```

### 2）编译lighty-core

```sh
cd lighty-core
mvn clean install -DskipTests
```

### 3）编译lighty

```sh
cd lighty
mvn clean install -DskipTests
```

### 4）运行lighty-transportpce

```sh
cd lighty/target
java -ms128m -mx128m -XX:MaxMetaspaceSize=128m -jar lighty-transportpce-11.0.1.jar
#或执行启动脚本start-controller.sh
cd lighty/target/assembly/resources
sh start-controller.sh
```

## ODL/Karaf vs lighty.io 启动性能对比

```txt
| Property Name                     | ODL/Karaf *    | lighty.io ** |
|-----------------------------------|----------------|--------------|
| Build size                        | 225M           | 64M          |
| Startup Time                      | ~15s           | ~6s          |
| Shutdown Time                     | ~5s            | ~100ms       |
| Process memory allocated (RSS)*** | 1236 MB        | 353 MB       |
| HEAP memory (used/allocated)      | 135 / 1008 MB  | 58 / 128 MB  |
| Metaspace (used/allocated)        | 115 / 132 MB   | 62 /  65 MB  |
| Threads (live/daemon)             | 111 / 48       | 70 /  11     |
| Classes loaded                    | 22027          | 12019        |
| No. of jars                       | 680            | 244          |
```

