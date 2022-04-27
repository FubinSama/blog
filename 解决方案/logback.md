# logback

## 基本配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">

    <property name="PATH" value="log/columbia/gateway" />
    <property name ="PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %highlight(%-5level) %cyan(%logger{32}):%line - %msg%n" />
    <property name="CHARSET" value="UTF-8" />

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${PATTERN}</pattern>
            <charset>${CHARSET}</charset>
        </encoder>
    </appender>

    <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${PATH}/info.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${PATH}/info.log.%d{yyyy-MM-dd}</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <pattern>${PATTERN}</pattern>
            <charset>${CHARSET}</charset>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <!--            <onMismatch>DENY</onMismatch>-->
        </filter>
    </appender>

    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${PATH}/error.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${PATH}/error.log.%d{yyyy-MM-dd}</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <pattern>${PATTERN}</pattern>
            <charset>${CHARSET}</charset>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <appender name="TASK_PULL_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${PATH}/task_pull.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${PATH}/task_pull.log.%d{yyyy-MM-dd}</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <pattern>${PATTERN}</pattern>
            <charset>${CHARSET}</charset>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
        </filter>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="INFO_FILE" />
        <appender-ref ref="ERROR_FILE" />
    </root>

    <!-- 不同的包，设置不同的日志等级 -->
    <logger name="com.mib.bank" level="DEBUG" />
    <logger name="org.apache.dubbo" level="ERROR" />
    <logger name="com.mib.bank.pay.columbia.front.task.BankColombiaPushTask" level="INFO" additivity="false">
        <appender-ref ref="TASK_PULL_FILE" />
    </logger>

</configuration>
```
