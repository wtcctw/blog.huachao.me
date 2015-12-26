title:  "slf4j基础"
date: 2015-04-19
updated: 2015-12-26
categories: 
- 后端
tags:
- slf4j
- log
---

> Java日志必备神器slf4j

## 我期望的日志系统
- 配置简单，写到文件、标准输出、标准错误，能有各种输出格式
- 延续Linux上log rotation的优良传统，肯定要能根据配置的时间自动进行backup
- 性能也是一个关键的考核指标，性能要是不好，谁都不会用
- 如果能像Linux上一样，简单配置后发送到远端，还能有自动分析的工具，就太好啦

## slf4j依赖包

- ch.qos.logback:logback-classic

## slf4j配置文件

``` xml
<configuration>
    <property name="appName" value="ibatis-basic" />
    <contextName>${appName}</contextName>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern> %d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} [method:%method] [line:%line] - %msg%n</pattern>
        </encoder>
    </appender>
    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>logFile.log</file>
        <append>true</append>
        <encoder>
            <pattern> %d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} [method:%method] [line:%line] - %msg%n</pattern>
        </encoder>
    </appender>
    <appender name="FILE.me.huachao" class="ch.qos.logback.core.FileAppender">
        <file>logFile.me.huachao.log</file>
        <append>true</append>
        <encoder>
            <pattern> %d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} [method:%method] [line:%line] - %msg%n</pattern>
        </encoder>
    </appender>
    <appender name="RollingFile" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logFile.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern> %d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} [method:%method] [line:%line] - %msg%n</pattern>
        </encoder>
    </appender>
    <logger name="me.huachao" level="DEBUG" additivity="false">
        <appender-ref ref="FILE.me.huachao"/>
    </logger>
    <root level="WARNING">
        <appender-ref ref="FILE"/>
        <appender-ref ref="RollingFile"/>
    </root>
</configuration>
```

*additivity：表示当前logger的信息是否需要在root中重现出来, true为需要重现，false为不需要*

## 实例代码
``` java
private static final Logger logger = LoggerFactory.getLogger(StudentService.class);
logger.debug("exception:", e);
```

## Code
[MySampleCode@Github](https://github.com/wtcctw/ibatis-basic)