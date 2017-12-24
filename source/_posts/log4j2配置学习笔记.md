---
title: log4j2配置学习笔记
date: 2017-03-02 19:55:27
tags: 
- 配置 
- log4j2
categories: 配置
---

根节点(1级): Configuraion

# 2级节点: 
# Appenders
常见三种: 

- Console : 控制台
- File: 指定位置的文件. fileName可带路径.
- RoolingFile: 超过指定大小自动删除旧的创建新的Appender. 
    - filePattern: 指定新建日志文件的名称格式.
    - Policies: 可以以时间滚动\文件大小滚动\日志文件数量.

# Loggers

常见两种:Root和Logger

- Root: 默认配置,Logger里没有的就会继承Root里的.
    - level: ALL<Trace<Debug<Info<Warn<Error<Fatal<OFF
    - AppenderRef: 具体输出到哪个Appender. 
- Logger: 比如为指定包下的class指定不同的日志级别.
    - name: 指定所适用的类或包的全路径.


# 异步logger
需要加入依赖:
```
 compile group: 'com.lmax', name: 'disruptor', version: '3.3.7'
```

`log4j2.component.properties`文件:
```
log4j2.AsyncQueueFullPolicy: Discard
#AsyncLoggerConfig.RingBufferSize: 524288
```

异步配置:
```
Configuration:
  properties:
    property:
      - name: logPath
        value: /Users/xiaoyue26/learn/logs/
      - name: filename
        value: dev.log
      - name: pattern
        value: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%p] [%t] [%c] @@@traceId=%X{TRACE_ID}@@@ %m%n"
      - name: httpRequestPattern
        value: "%-d{yyyy-MM-dd HH:mm:ss.SSS} @@@traceId=%X{TRACE_ID}@@@ %m%n"

  status: "info"
  Appenders:
    RollingRandomAccessFile:
      - name: "FileAppender"
        fileName: "${logPath}${filename}"
        filePattern: "${logPath}${filename}.%d{yyyy-MM-dd}"
        PatternLayout:
          pattern: "${pattern}"
        Policies:
          TimeBasedTriggeringPolicy: {}
        immediateFlush: false
      - name: "AnalysisFileAppender"
        fileName: "${logPath}analysis-${filename}"
        filePattern: "${logPath}analysis-${filename}.%d{yyyy-MM-dd}"
        PatternLayout:
          pattern: "${pattern}"
        Policies:
          TimeBasedTriggeringPolicy: {}
        immediateFlush: false
      - name: "HTTPRequestFileAppender"
        fileName: "${logPath}http-request-${filename}"
        filePattern: "${logPath}http-request-${filename}.%d{yyyy-MM-dd}"
        PatternLayout:
          pattern: "${httpRequestPattern}"
        Policies:
          TimeBasedTriggeringPolicy: {}
        immediateFlush: false
      - name: "RPCRequestFileAppender"
        fileName: "${logPath}rpc-request-${filename}"
        filePattern: "${logPath}rpc-request-${filename}.%d{yyyy-MM-dd}"
        PatternLayout:
          pattern: "${pattern}"
        Policies:
          TimeBasedTriggeringPolicy: {}
        immediateFlush: false
    Async:
      - name: "AsyncFileAppender"
        AppenderRef:
          - ref: FileAppender
        bufferSize: 10000
      - name: "AsyncAnalysisFileAppender"
        AppenderRef:
          - ref: AnalysisFileAppender
        bufferSize: 15000
      - name: "AsyncHTTPRequestFileAppender"
        AppenderRef:
          - ref: HTTPRequestFileAppender
        bufferSize: 10000
      - name: "AsyncRPCRequestFileAppender"
        AppenderRef:
          - ref: RPCRequestFileAppender
        bufferSize: 10000

  Loggers:
    AsyncLogger:
      - name: "RequestLogger"
        level: info
        additivity: false
        AppenderRef:
          - ref: AsyncHTTPRequestFileAppender
      - name: "RpcRequestLogger"
        level: info
        additivity: false
        AppenderRef:
          - ref: AsyncRPCRequestFileAppender
      - name: "AnalysisLogger"
        level: info
        additivity: false
        AppenderRef:
          - ref: AsyncAnalysisFileAppender
    AsyncRoot:
      level: info
      AppenderRef:
        - ref: AsyncFileAppender


```


示例同步配置`yml`:
```
Configutation:
  status: warn

  Appenders:
    Console: # 控制台的appender
      name: CONSOLE
      target: SYSTEM_OUT
      PatternLayout:
        Pattern: "%d{ISO8601} %-5p [%c{3}] [%t] %m%n"
    RollingFile: # 输出到文件,超过的归档
      - name: APPLICATION
        fileName: /Users/xiaoyue26/learn/logs/test.log
        filePattern: "/Users/xiaoyue26/learn/logs/$${date:yyyy-MM}/test-%d{yyyy-MM-dd}-%i.log.gz"
        PatternLayout:
          Pattern: "%d{ISO8601} %-5p [%c{3}] [%t] %m%n"
        policies:
          TimeBasedTriggeringPolicy:
            interval: 1 # 1小时
            modulate: true # 自动对齐时间间隙

  Loggers:
      Root:
        level: info
        AppenderRef:
          - ref: CONSOLE
          - ref: APPLICATION
      Logger:
        - name: com.myco.myapp.Foo
          additivity: false
          level: info
          AppenderRef:
            - ref: CONSOLE
            - ref: APPLICATION
        - name: com.myco.myapp.Bar
          additivity: false # 不输出到Root的appenderRef里了
          level: debug
          AppenderRef:
            - ref: CONSOLE
            - ref: APPLICATION
```


