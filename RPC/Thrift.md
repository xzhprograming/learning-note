## Thrift

### 简介

Thrift是一个轻量级、跨语言的远程服务调用框架，最初由Facebook开发，后面进入Apache开源项目。它通过自身的IDL中间语言, 并借助代码生成引擎生成各种主流语言的RPC服务端/客户端模板代码。

Thrift支持多种不同的编程语言，包括C++、Java、Python、PHP、Ruby。

可以实现Java客户端调用Python服务端、Python客户端调用Java服务端(其他语言之间的跨语言调用和这个类似)、Java客户端调用Java服务端、Python客户端调用Python服务端，特别是跨语言异构平台之间的调用价值很大，且比基于HTTP调用方式的RPC框架效率高很多。

Thrift的主要特点有：

1. 基于二进制的高性能的编解码框架
2. 基于NIO的底层通信
3. 相对简单的服务调用模型
4. 使用IDL支持跨平台调用

### 安装



