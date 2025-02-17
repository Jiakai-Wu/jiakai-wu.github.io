---
title: Service Provider Interface(SPI)
date: 2023-02-05 20:14:15 +0800 
categories: [design_patterns, example]
tags: [interface, design_patterns] 
---
### SPI (Service Provider Interface)

#### Concept And Examples

**Service provider interface**(**SPI**) is an [API](https://en.wikipedia.org/wiki/Application_programming_interface) intended to be implemented or extended by a third party. It can be used to enable framework extension and replaceable components.

A service is a well-known set of interfaces and (usually abstract) classes. A service provider is a specific implementation of a service. The classes in a provider typically implement the interfaces and subclass the classes defined in the service itself. Service providers can be installed in an implementation of the Java platform in the form of extensions, that is, jar files placed into any of the usual extension directories. Providers can also be made available by adding them to the application's class path or by some other platform-specific means.

The concept can be extended to other platforms using the corresponding tools. In the [Java Runtime Environment](https://en.wikipedia.org/wiki/Java_Runtime_Environment), SPIs are used in:

- [Java Database Connectivity](https://en.wikipedia.org/wiki/Java_Database_Connectivity)
- [Java Cryptography Extension](https://en.wikipedia.org/wiki/Java_Cryptography_Extension)
- [Java Naming and Directory Interface](https://en.wikipedia.org/wiki/Java_Naming_and_Directory_Interface)
- [Java API for XML Processing](https://en.wikipedia.org/wiki/Java_API_for_XML_Processing)
- [Java Business Integration](https://en.wikipedia.org/wiki/Java_Business_Integration)
- Java Sound
- Java Image I/O
- Java File Systems

#### SPI 两种典型实现

- jdk spi ： 目录在 /META-INF/services/资源文件夹下，文件名为 jdk 中接口，文件内容为 实现的接口类全路径
- spring spi:   目录在 /META-INF/spring.factories，SpringFactoriesClassLoader类中的loadFactories方法来加载

##### Example  (Java Database Connectivity  in JRE)

java jdk中的类

![javaJDK](/assets/img/2023-02-05-SPI(Service_Provider_Interface)/javaJDK.png)

具体实现者配置，即一层映射关系，将Driver接口 的具体实现类映射到三方具体实现

![映射关系](/assets/img/2023-02-05-SPI(Service_Provider_Interface)/mapping.png)

具体三方实现类

![具体实现类](/assets/img/2023-02-05-SPI(Service_Provider_Interface)/class.png)

##### Spring starter 写法 也是类似原理
