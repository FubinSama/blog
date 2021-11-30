# SpringBootTest解决方案

## 想要使用`src/main/resources`下的配置

```XML
<build>
<!-- 让单元测试使用`src/main/resources`下的配置文件 -->
    <testResources>
        <testResource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
            <includes>
                <include>*.properties</include>
                <include>*.xml</include>
                <include>i18n/*</include>
            </includes>
        </testResource>
    </testResources>
</build>
```

## 配置使用上下文

```java
@SpringBootApplication(scanBasePackages = ["想要测试的组件所在的包路径"])
@PropertySource(value = ["想要加载使用的配置文件路径，如：", "classpath:/application-service.properties"], encoding = "UTF-8")
@FixMethodOrder(MethodSorters.JVM) // 配置使用代码出现的顺序作为测试的执行顺序
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE, classes = [KenyaMapperTestsConfig::class])
@RunWith(SpringRunner::class)
```

## java.util.Date和java.sql.Date

注意将`java.util.Date`类型存入数据库，再取出时会导致其`fastTime`字段丢失精度（从13位的微秒变成只到10位的毫秒加上3个0），而其`equals`方法是比较的`fastTime`是否相等，因此`Assert.assertEquals`断言会失败。为此，推荐在单元测试中使用`java.util.Date`时，将其精度变为毫秒级(四舍五入)，如下所示：

```java
private fun Date.fixPrecision() = this.apply { this.time = (this.time / 1_000L) * 1_000L }
val now = Date().fixPrecision()
```

## java.math.BigDecimal

BigDecimal的`equals`方法要求必须精度相同，也就是说`0.00`和`0`是不相等的。

## 想要回滚测试单元

```java
@Test
@Transactional
@Rollback(true)
fun test() {

}
```
