---
title: 如何快速编译构建ODL项目
date: 2020-05-05
category: OpenDaylight
tags: odl
description: ODL项目的多模块化导致各模块之间的依赖性很强，在编译ODL项目的时候，往往会耗费很长的时间，这对于开发人员在开发调试时是非常不友好的。本文简单从构建策略和构建参数入手，分析不同构建场景下的最佳构建命令，从而加快ODL项目的构建速度，提高开发效率。
---

## 构建策略profile

odl项目总工程的根目录pom.xml文件中的profiles标签添加如下profile构建策略，构建时使用-P<id>激活策略

```xml
<profile>
    <id>q</id>
    <properties>
        <skipITs>true</skipITs>
        <skip.karaf.featureTest>true</skip.karaf.featureTest>
        <jacoco.skip>true</jacoco.skip>
        <maven.javadoc.skip>true</maven.javadoc.skip>
        <maven.source.skip>true</maven.source.skip>
        <checkstyle.skip>true</checkstyle.skip>
        <findbugs.skip>true</findbugs.skip>
        <pmd.skip>true</pmd.skip>
        <cpd.skip>true</cpd.skip>
        <maven.site.skip>true</maven.site.skip>
        <invoker.skip>true</invoker.skip>
        <enforcer.skip>true</enforcer.skip>
        <mdsal.skip.verbose>true</mdsal.skip.verbose>
        <gitid.skip>true</gitid.skip>
        <skipTests>true</skipTests>
        <maven.test.skip>true</maven.test.skip> <!-- 编译test包请注释改项 -->
    </properties>
</profile>
```

## odl项目构建说明

1）首次构建，使用全量构建

```sh
mvn clean install -Pq
```

2）首次全量构建正常的前提下，修改部分模块后，使用增量构建

```sh
# xxxx为修改的模块名；修改多个模块时，xxxx为多个模块的公用父模块
# "-pl xxxx -amd" 表示级联构建依赖xxxx的模块
mvn clean install -pl xxxx -amd -Pq
```

3）如果构建过程中yyyy模块构建失败，使用参数-rf <模块名>可指定从构建失败的模块yyyy重新开始构建

```sh
# 全量构建失败时使用
mvn clean install -rf yyyy -Pq
# 增量构建失败时使用
mvn clean install -pl xxxx -amd -rf yyyy -Pq
```

## 构建实例参考

```sh
mvn clean install -Pq

[INFO] Reactor Build Order:
[INFO]
[INFO] transportpce-artifacts                                             [pom]
[INFO] transportpce-ordmodels-common                                   [bundle]
[INFO] transportpce-ordmodels-device                                   [bundle]
[INFO] transportpce-ordmodels-network                                  [bundle]
[INFO] transportpce-ordmodels-service                                  [bundle]
[INFO] transportpce-ordmodels-r2000                                    [bundle]
[INFO] transportpce-ordmodels                                             [pom]
[INFO] transportpce-api                                                [bundle]
[INFO] transportpce-common                                             [bundle]
[INFO] transportpce-olm                                                [bundle]
[INFO] transportpce-renderer                                           [bundle]
[INFO] transportpce-networkmodel                                       [bundle]
[INFO] transportpce-pce                                                [bundle]
[INFO] transportpce-servicehandler                                     [bundle]
[INFO] transportpce-impl                                               [bundle]
[INFO] OpenDaylight :: transportpce :: ordmodels                      [feature]
[INFO] OpenDaylight :: transportpce :: api                            [feature]
[INFO] OpenDaylight :: transportpce                                   [feature]
[INFO] features-transportpce                                          [feature]
[INFO] features-aggregator                                                [pom]
[INFO] transportpce-karaf                                                 [pom]
[INFO] transportpce                                                       [pom]
```

```sh
mvn clean install -rf common -Pq

[INFO] Reactor Build Order:
[INFO]
[INFO] transportpce-common                                             [bundle]
[INFO] transportpce-olm                                                [bundle]
[INFO] transportpce-renderer                                           [bundle]
[INFO] transportpce-networkmodel                                       [bundle]
[INFO] transportpce-pce                                                [bundle]
[INFO] transportpce-servicehandler                                     [bundle]
[INFO] transportpce-impl                                               [bundle]
[INFO] OpenDaylight :: transportpce :: ordmodels                      [feature]
[INFO] OpenDaylight :: transportpce :: api                            [feature]
[INFO] OpenDaylight :: transportpce                                   [feature]
[INFO] features-transportpce                                          [feature]
[INFO] features-aggregator                                                [pom]
[INFO] transportpce-karaf                                                 [pom]
[INFO] transportpce                                                       [pom]
```

```sh
mvn clean install -pl common -amd -Pq

[INFO] Reactor Build Order:
[INFO]
[INFO] transportpce-common                                             [bundle]
[INFO] transportpce-olm                                                [bundle]
[INFO] transportpce-renderer                                           [bundle]
[INFO] transportpce-networkmodel                                       [bundle]
[INFO] transportpce-pce                                                [bundle]
[INFO] transportpce-servicehandler                                     [bundle]
[INFO] OpenDaylight :: transportpce                                   [feature]
[INFO] features-transportpce                                          [feature]
[INFO] transportpce-karaf                                                 [pom]
```

```sh
mvn clean install -pl common -amd -rf servicehandler -Pq

[INFO] Reactor Build Order:
[INFO]
[INFO] transportpce-servicehandler                                     [bundle]
[INFO] OpenDaylight :: transportpce                                   [feature]
[INFO] features-transportpce                                          [feature]
[INFO] transportpce-karaf                                                 [pom]
```

## 常用构建参数

```sh
参数		说明
-pl		选项后可跟随{groupId}:{artifactId}或者所选模块的相对路径(多个模块以逗号分隔)
-am		表示同时处理选定模块所依赖的模块
-amd	表示同时处理依赖选定模块的模块
-N		表示不递归子模块
-rf		表示从指定模块开始继续处理
```