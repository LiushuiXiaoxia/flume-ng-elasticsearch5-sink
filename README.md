# Flume 1.6.0 supporting ElasticSearch 5.0
Base on: https://github.com/arberzal/flume-ng-elasticsearch2-sink

This version of the Flume ElasticSearch sink works with ElasticSearch 5.0
but to do so, **some libraries used by Flume had to be updated to the newest
versions**,  **For this reason, many tests do not work retuning a pretty "jar
hell" exception.**

As Flume is packaged right now (sink and flume libraries in the  same class 
loader) seems difficult to find another solution. The best way, without
modifying Flume, would be a new ElasticSearch sink based on http API and 
built on top of a reactive framework.


**问题描述**

服务器使用ELK来收集日志，需要批量发送数据，有时候会出现一批数据中，只有部分数据发送成功，结果没有明确给出哪条数据成功或者失败。所以需要重复发送，这样就会导致原先已经发送成功的数据存在重复问题。

**解决方案**

调研得出，每次发送数据，ElasticSearch不检查数据是否存在，直接保存和插入的，我们可以在原先的数据中申明本条数据的主键ID，那么ElasticSearch自己就不会自动生成ID，而使用数据中已经存在的ID，然后发送重复数据的时候，就会先校验数据是否已经存在，然后再插入。
那么问题来了，如果自己生成ID呢，并且保证同一条数据每次生成的ID都是一样的，所以考虑使用Hash的方法，最终选定使用sha1算法。考虑到APM数据中含有时间错和客户端的UUID，这样每条数据基本上不会出现重复问题，并且ElasticSearch中数据只会保存三天时间，就算有重复概率，也是比较小的，综合考虑利大于弊。

## Modifications to your Flume lib path:
    
### Libraries to be added:
##### this project
- flume-ng-elasticsearch5-sink-1.6.0.jar

##### common library
- t-digest-3.0.jar
- guava-19.0.jar
- hppc-0.7.2.jar
- httpclient-4.2.1.jar
- httpcore-4.2.1.jar
- jackson-annotations-2.8.1.jar
- jackson-core-2.8.1.jar
- jackson-databind-2.8.1.jar
- jackson-dataformat-cbor-2.8.1.jar
- jackson-dataformat-smile-2.8.1.jar
- jackson-dataformat-yaml-2.8.1.jar
- joda-convert-1.2.jar
- joda-time-2.9.4.jar
- snakeyaml-1.17.jar

##### elasticsearch-5.0, plugins & dependencies
- elasticsearch-5.0.0.jar
- lang-mustache-client-5.0.0.jar
- percolator-client-5.0.0.jar
- reindex-client-5.0.0.jar
- lucene-analyzers-common-6.2.0.jar
- lucene-backward-codecs-6.2.0.jar
- lucene-core-6.2.0.jar
- lucene-grouping-6.2.0.jar
- lucene-highlighter-6.2.0.jar
- lucene-join-6.2.0.jar
- lucene-memory-6.2.0.jar
- lucene-misc-6.2.0.jar
- lucene-queries-6.2.0.jar
- lucene-queryparser-6.2.0.jar
- lucene-sandbox-6.2.0.jar
- lucene-spatial-6.2.0.jar
- lucene-suggest-6.2.0.jar

##### transportClient dependencies
- transport-5.0.0.jar
- transport-netty3-client-5.0.0.jar
- transport-netty4-client-5.0.0.jar
- netty-3.10.6.Final.jar
- netty-buffer-4.1.5.Final.jar
- netty-codec-4.1.5.Final.jar
- netty-common-4.1.5.Final.jar
- netty-resolver-4.1.5.Final.jar
- netty-transport-4.1.5.Final.jar

##### log4j2 dependencies
- log4j-1.2.16.jar
- log4j-1.2-api-2.6.2.jar
- log4j-api-2.6.2.jar
- log4j-core-2.6.2.jar
- log4j-slf4j-impl-2.1.jar

#### avro dependencies
- jackson-mapper-asl-1.8.8.jar
- jackson-core-asl-1.8.8.jar 
- retrotranslator-runtime-1.2.9.jar
- backport-util-concurrent-3.1.jar
- paranamer-2.3.jar
- snappy-java-1.1.0.jar
- commons-compress-1.4.1.jar

### Libraries to be removed:
- jackson-annotations-2.3.0.jar
- jackson-core-2.3.1.jar
- jackson-databind-2.3.1.jar
- joda-time-2.1.jar
- log4j-1.2.17.jar
- netty-3.5.12.Final.jar
- guava-11.0.2.jar
- flume-ng-elasticsearch-sink-1.6.0.jar
- elasticsearch-1.7.4.jar
- lucene-core-4.10.4.jar
- slf4j-log4j12-1.6.1.jar

## Upgrade log conf from log4j to log4j2
```xml
<?xml version="1.0" encoding="UTF-8"?>
    <configuration status="error" >
        <appenders>
            <Console name="Console" target="SYSTEM_OUT">
                <ThresholdFilter level="trace" onMatch="ACCEPT" onMismatch="DENY"/>
                <PatternLayout pattern="%d{HH:mm:ss.SSS} %-5level %class{36} %L %M - %msg%xEx%n"/>
            </Console>

             <RollingFile name="RollingFile" fileName="logs/flume.log"
                     filePattern="logs/flume.%d{yyyy-MM-dd}-%i.log"
                     ignoreExceptions="false">
                <PatternLayout charset="UTF-8" pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} %L %M - %msg%xEx%n" />
                <Policies>
                    <TimeBasedTriggeringPolicy />
                    <SizeBasedTriggeringPolicy size="100 MB"/>
                </Policies>
            </RollingFile>
        </appenders>

        <loggers>
            <root level="info">
                <appender-ref ref="RollingFile"/>
                <appender-ref ref="Console"/>
            </root>
        </loggers>
    </configuration>
```
