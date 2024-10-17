
---
title: "OpenIM源码分析: magefile编译和运行"
date: 2024-08-05T11:12:15+08:00
draft: false
categories:
 - open-im-server
tags:
 - golang
---


##  build

- 检查start-config.yaml是否存在, 并读取其中的内容
  
初始化结构体
  ```go
    type Config struct {
        ServiceBinaries    map[string]int `yaml:"serviceBinaries"`
        ToolBinaries       []string       `yaml:"toolBinaries"`
        MaxFileDescriptors int            `yaml:"maxFileDescriptors"`
    }

  ```

- mageutil init()根据当前目录设置创建如下目录
  + _output/bin/platforms/os/arch/
  + _output/bin/tools/os/arch/
  + _output/logs
  + _output/tmp
  + _output/tools
  
- 检查platform，编译
  + 遍历并编译当前目录下cmd下的main.go
  + 遍历并编译当前目录下tools下的main.go
  
## 启动

- 首先启动tools，如果其中某个tool启动失败，直接退出
  1. check-free-memory
    检查可用内存是否小于1GB
  2. check-component
    首先从flag -c 获取配置目录，根据配置文件名常量，加载配置文件，初始化配置后，
    创建客户端依次连接检查mongo, redis, kafka, minio（如果有配置的话）, etcd/zookeeper
    只有全部组件检查成功才返回成功
  3. seq 
     检查mongodb中data_version集合中 {key:seq, value: xx}中的value值与38常量比较
     如果大于等于32，表示同步过，不做处理，如果小于，那么将同步redis中的数据到mongodb中
     主要是以下几个prefix开头的数据:
     - MAX_SEQ:
     - MIN_SEQ:
     - CON_USER_MIN_SEQ:
     - HAS_READ_SEQ:
- 杀死已经存在的服务进程
- 检查是否已经全部杀死
- 依次启动start-config.yml中serviceBinaries中的服务
  传入参数-i {0-进程数} -c cwd/config
- 检查服务是否都启动成功，如果都成功，打印进程监听端口
