# springboot metrics

## micrometer

> `https://micrometer.io/docs`

### 监控系统的三个重要特性

- Dimensionality：系统是否支持使用标签键/值对丰富指标名称。如果系统不是dimensional的，那么它是hierarchical的，这意味着它只支持平面度量名称。将指标发布到分层系统时，Micrometer会展平标签键/值对集并将它们添加到名称中。
- Rate Aggregation：在规定的时间间隔内聚合一组样本。一些监控系统期望某些类型的离散样本（例如计数）在发布之前由应用程序转换为比率。其他系统期望始终发送累积值。还有一些人对此没有任何意见。
- Publishing：一些系统希望在空闲时轮询应用程序以获取指标，而其他系统则希望指标定期推送给它们。

### Registry - 指标注册器

`Meter`是用于收集有关您的应用程序的一组测量值（我们单独称为指标）的接口。`Micrometer`中的`Meter`是创建并保存在`MeterRegistry`中的。每个支持的监控系统都有一个 `MeterRegistry`的实现。

`Micrometer`包括一个`SimpleMeterRegistry`，它在内存中保存每个仪表的最新值，并且不会将数据导出到任何地方。在基于spring的应用中，会自动注入该类。

`Micrometer`提供了一个`CompositeMeterRegistry`，您可以向其中添加多个registries，让您同时将指标发布到多个监控系统。

```java
CompositeMeterRegistry composite = new CompositeMeterRegistry();
Counter compositeCounter = composite.counter("counter");
compositeCounter.increment(); // Increments are NOOP’d until there is a registry in the composite. The counter’s count still yields 0 at this point.
SimpleMeterRegistry simple = new SimpleMeterRegistry();
composite.add(simple); // A counter named counter is registered to the simple registry.
compositeCounter.increment(); // The simple registry counter is incremented, along with counters for any other registries in the composite.
```

另外`globalRegistry`类内置了一个静态的`globalRegistry`属性，它的类型是`CompositeMeterRegistry`。

### Meters

`Micrometer`支持一组`Meter`原语，包括`Timer`、`Counter`、`Gauge`、`DistributionSummary`、`LongTaskTimer`、`FunctionCounter`、`FunctionTimer`和`TimeGauge`。

`meter`由其`name`和`dimensions`唯一标识。`dimensions`也可以叫做`tags`，`Micrometer`接口使用的`tag`，因为它更短。一般来说，应该使用`name`作为枢轴，使用`dimensions`让一个特定的命名指标be sliced to drill down and reason about the data。这意味着，如果仅选择了`name`，您可以使用其他`dimensions` drill down and reason about the value being shown。

### Naming Meters - 指标命名规范

`Micrometer`采用命名约定，用`.`分隔小写单词。监控系统的每个`Micrometer`实现都带有一个命名约定，将小写点符号名称转换为监控系统推荐的命名约定。您可以通过实现 `NamingConvention`并将其设置在注册表上来覆盖注册表的默认命名约定：

```java
registry.config().namingConvention(myCustomNamingConvention);
```

通过适当的命名约定，以下在`Micrometer`中注册的计时器在各种监控系统中会转化为不同的名字：

```java
registry.timer("http.server.requests");
```

1. Prometheus：`http_server_requests_duration_seconds`
2. Atlas：`httpServerRequests`
3. Graphite：`http.server.requests`
4. InfluxDB：`http_server_requests`

通过遵守`Micrometer`的小写点符号约定，您可以保证您的指标名称在监控系统中具有最大程度的可移植性。

#### Tag Naming - tag命名规范

> 推荐在命名`Tag`时同样遵循小写点表示法。使用这种一致的标签命名约定可以更好地转换为相应监控系统的惯用命名方案。

##### 推荐的实现

```java
// public Counter counter(String name, String... tags);
registry.counter("database.calls", "db", "users")
registry.counter("http.requests", "uri", "/api/users")
```

这种命名提供了足够的上下文，只靠`name`就可以知道该指标潜在的意义。例如：如果我们选择`database.calls`，我们可以看到对所有数据库的调用总数。然后我们可以通过`group by`或 `select by` `db`进一步`drill down`或对每个数据库的调用进行比较分析。

##### 坏的实现

```java
// public Counter counter(String name, String... tags);
registry.counter("calls", "class", "database", "db", "users");
registry.counter("calls", "class", "http", "uri", "/api/users");
```

在这种方法中，如果我们选择`calls`，我们会得到一个值，该值是对`db`和`API`端点的调用次数的总和。如果没有进一步的`dimensional drill-down`，这个时间序列就没有实际意义了。

#### Common Tags - 通用的全局tags

您可以在注册表级别定义通用`Tag`，并将它们添加到报告给监控系统的每个指标中。这通常用于对操作环境进行`dimensional drill-down`，例如主机、实例、区域、堆栈等：

```java
registry.config().commonTags("stack", "prod", "region", "us-east-1");
registry.config().commonTags(Arrays.asList(Tag.of("stack", "prod"), Tag.of("region", "us-east-1"))); // equivalently
```

> 公共标签通常必须在任何（可能自动配置的）仪表绑定器之前添加到注册表中。根据您的环境，有不同的方法可以实现此目的。
> 如果你使用 Spring Boot，你有两个选择：
>
> 1. Add your common tags with configuration properties.
> 2. If you need more flexibility (for example, you have to add common tags to a registry defined in a shared library), register a `MeterRegistryCustomizer` callback interface as a bean to add your common tags.

#### Tag Values - tag值的规定

Tag values must be non-null.

> 考虑用于记录服务端点上的 HTTP 请求的 URI 标记。如果我们不将 404 限制为像 NOT_FOUND 这样的值，则指标的维度将随着找不到的每个资源而增加。

### Meter Filters - 指标过滤器

您可以使用仪表过滤器配置每个注册表，这使您可以更好地控制仪表的注册方式和时间以及它们发出的统计数据类型。 仪表滤波器具有三个基本功能：

1. `Deny` (or `Accept`) meters being registered.
2. `Transform` meter IDs (for example, changing the name, adding or removing tags, and changing description or base units).
3. `Configure` distribution statistics for some meter types.

`MeterFilter`的实现以编程方式添加到注册表中：

```java
registry.config()
    .meterFilter(MeterFilter.ignoreTags("too.much.information"))
    .meterFilter(MeterFilter.denyNameStartsWith("jvm"));
```

#### Deny or Accept Meters - 通过过滤器选择激活或禁止哪些指标监听

```java
new MeterFilter() {
    @Override
    public MeterFilterReply accept(Meter.Id id) {
       if(id.getName().contains("test")) {
          return MeterFilterReply.DENY;
       }
       return MeterFilterReply.NEUTRAL;
    }
}
```

`MeterFilterReply`具有三种可能的状态：

1. `DENY`：不要让这个`meter`被注册。当您尝试针对注册表注册一个`meter`并且过滤器返回`DENY`时，注册表将返回该`meter`的`NOOP`版本（例如，`NoopCounter`或`NoopTimer`）。您的代码可以继续与`NOOP` `meter`交互，但记录到它的任何内容都会立即丢弃，开销很小。
2. `NEUTRAL`：如果没有其他`meter`过滤器返回`DENY`，仪表注册将正常进行。
3. `ACCEPT`：如果过滤器返回`ACCEPT`，则立即注册`meter`，而无需询问任何其他过滤器的`ACCEPT`方法。

##### Convenience Methods - 通用的过滤器方法

```java
public interface MeterFilter {
    // accept
    static MeterFilter accept() {
        return MeterFilter.accept(id -> true);
    }
    static MeterFilter acceptNameStartsWith(String prefix) {
        return accept(id -> id.getName().startsWith(prefix));
    }
    static MeterFilter accept(Predicate<Meter.Id> iff) {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return iff.test(id) ? MeterFilterReply.ACCEPT : MeterFilterReply.NEUTRAL;
            }
        };
    }
    // deny
    static MeterFilter deny() {
        return MeterFilter.deny(id -> true);
    }
    static MeterFilter denyNameStartsWith(String prefix) {
        return deny(id -> id.getName().startsWith(prefix));
    }
    static MeterFilter deny(Predicate<Meter.Id> iff) {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return iff.test(id) ? MeterFilterReply.DENY : MeterFilterReply.NEUTRAL;
            }
        };
    }
    static MeterFilter maximumAllowableMetrics(int maximumTimeSeries) {
        return new MeterFilter() {
            private final Set<Meter.Id> ids = ConcurrentHashMap.newKeySet();

            @Override
            public MeterFilterReply accept(Meter.Id id) {
                if (ids.size() > maximumTimeSeries) return MeterFilterReply.DENY;
                ids.add(id);
                return ids.size() > maximumTimeSeries ? MeterFilterReply.DENY : MeterFilterReply.NEUTRAL;
            }
        };
    }
    static MeterFilter maximumAllowableTags(String meterNamePrefix, String tagKey, int maximumTagValues, MeterFilter onMaxReached) {
        return new MeterFilter() {
            private final Set<String> observedTagValues = ConcurrentHashMap.newKeySet();

            @Override
            public MeterFilterReply accept(Meter.Id id) {
                String value = matchNameAndGetTagValue(id);
                if (value != null) {
                    if (!observedTagValues.contains(value)) {
                        if (observedTagValues.size() >= maximumTagValues) {
                            return onMaxReached.accept(id);
                        }
                        observedTagValues.add(value);
                    }
                }
                return MeterFilterReply.NEUTRAL;
            }

            @Nullable
            private String matchNameAndGetTagValue(Meter.Id id) {
                return id.getName().startsWith(meterNamePrefix) ? id.getTag(tagKey) : null;
            }

            @Override
            public DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
                String value = matchNameAndGetTagValue(id);
                if (value != null) {
                    if (!observedTagValues.contains(value)) {
                        if (observedTagValues.size() >= maximumTagValues) {
                            return onMaxReached.configure(id, config);
                        }
                    }
                }
                return config;
            }
        };
    }
    static MeterFilter denyUnless(Predicate<Meter.Id> iff) {
        return new MeterFilter() {
            @Override
            public MeterFilterReply accept(Meter.Id id) {
                return iff.test(id) ? MeterFilterReply.NEUTRAL : MeterFilterReply.DENY;
            }
        };
    }
}
```

##### Chaining Deny Accept Meters - 链式组合生成黑白名单（过滤器）

```java
registry.config()
    .meterFilter(MeterFilter.acceptNameStartsWith("http"))
    .meterFilter(MeterFilter.deny());
```

这通过将两个过滤器堆叠在一起实现了另一种形式的白名单。此注册表中只允许存在`http`指标。

#### Transforming metrics - 将一个指标转化成另一个指标

```java
new MeterFilter() {
    @Override
    public Meter.Id map(Meter.Id id) {
       if(id.getName().startsWith("test")) {
          return id.withName("extra." + id.getName()).withTag("extra.tag", "value");
       }
       return id;
    }
}
```

#### Configuring Distribution Statistics - 可配置的分布统计

`Timer`和`DistributionSummary`包含一组可选的分布统计信息（不包含基本的计数、总数和最大值），您可以通过过滤器进行配置。这些分布统计数据包括预先计算的百分位数(percentiles)、服务质量/水平目标(SLOs)和直方图(histograms)。

```java
new MeterFilter() {
    @Override
    public DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
        if (id.getName().startsWith(prefix)) {
            return DistributionStatisticConfig.builder()
                    .publishPercentiles(0.9, 0.95)
                    .build()
                    .merge(config);
        }
        return config;
    }
};
```

### Rate Aggregation - rate聚合的方式

`Micrometer`知道特定的监控系统是否希望在发布指标之前在客户端进行速率聚合，它根据监控系统期望的风格来累积指标。

#### 服务端

这时客户端如实上报原数据，服务端在会自动进行数据修复和聚合计算。比如，上报的每秒http请求数序列为：`0 2 4 6 (宕机重启) 0 2 4 6`，则服务端给出的序列为：`0 2 4 6 6 8 10 12`，保证counter单调递增，并在调用rate函数时进行聚合计算。

#### 客户端

Um...数学容易头大，知道有个这玩意就是了。

### Counters

`Counter`接口允许您增加固定数量，该数量必须为正数。

> 永远不要计算您可以使用`Timer`计时或使用`DistributionSummary`进行汇总的内容！`Timer`和`DistributionSummary `是发布事件计数。

当从计数器构建图表和警报时，您通常应该最感兴趣的是测量某个事件在给定时间间隔内发生的速率。考虑一个简单的队列。您可以使用计数器来测量各种事物，例如插入和移除项目的速率。

```java
Normal rand = ...; // a random generator

MeterRegistry registry = ...;
Counter counter = registry.counter("counter"); // 注册计数器，名为counter

Flux.interval(Duration.ofMillis(10))
        .doOnEach(d -> {
            if (rand.nextDouble() + 0.1 > 0) {
                counter.increment(); // 如果满足条件，则计数+1
            }
        })
        .blockLast();
```

#### Function-tracking Counters

`FunctionCounter`实际上是加了个cache参数，感觉用不到。

### Gauges - 仪表

Gauges是获取当前值的句柄。Gauges的典型示例是集合或映射的大小或处于运行状态的线程数。

> 仪表对于监视具有自然上限的事物很有用。我们不建议使用仪表来监控诸如请求计数之类的东西，因为它们可以在应用程序实例的生命周期内无限增长。
> 可以用Counter计数的东西不要使用Gauges

```java
List<String> list = registry.gauge("listGauge", Collections.emptyList(), new ArrayList<>(), List::size); // 一种稍微更常见的仪表形式是监视某些非数字对象的仪表。最后一个参数建立用于在观察仪表时确定仪表值的函数。
List<String> list2 = registry.gaugeCollectionSize("listSize2", Tags.empty(), new ArrayList<>()); // 当您想要监视集合大小时，这是一种更方便的形式。
Map<String, Integer> map = registry.gaugeMapSize("mapGauge", Tags.empty(), new HashMap<>());
```

创建仪表的所有不同形式都只维护对被观察对象的弱引用，以免阻止对象的垃圾收集。

#### 手动增加或减少仪表

可以制作一个仪表来跟踪任何可设置的`java.lang.Number`子类型，例如`java.util.concurrent.atomic`中的`AtomicInteger`和`AtomicLong`以及类似的类型，例如`Guava`的`AtomicDouble`：

```java
// maintain a reference to myGauge
AtomicInteger myGauge = registry.gauge("numberGauge", new AtomicInteger(0));
// ... elsewhere you can update the value it holds using the object reference
myGauge.set(27);
myGauge.set(11);
```

#### Gauge Fluent Builder - 链式创建Gauge的示例

```java
Gauge gauge = Gauge
    .builder("gauge", myObj, myObj::gaugeValue)
    .description("a description of what this gauge does") // optional
    .tags("region", "test") // optional
    .register(registry);
```

#### Why is My Gauge Reporting NaN or Disappearing?

`Micrometer`不会对state object进行强饮用，所以要自己保证强饮用。如果状态对象被GC了，就会出现NaN的情况。

#### TimeGauge

`TimeGauge`是一种专门跟踪时间值的计量器，可缩放到每个注册表实现所期望的基本时间单位

```java
AtomicInteger msTimeGauge = new AtomicInteger(4000);
AtomicInteger usTimeGauge = new AtomicInteger(4000);
TimeGauge.builder("my.gauge", msTimeGauge, TimeUnit.MILLISECONDS, AtomicInteger::get).register(registry);
TimeGauge.builder("my.other.gauge", usTimeGauge, TimeUnit.MICROSECONDS, AtomicInteger::get).register(registry);
```

如果`registry`是`Prometheus`，它们将在几秒钟内转换如下：

```shell
# HELP my_gauge_seconds
# TYPE my_gauge_seconds gauge
my_gauge_seconds 4.0
# HELP my_other_gauge_seconds
# TYPE my_other_gauge_seconds gauge
my_other_gauge_seconds 0.004
```

#### Multi-gauge

```java
// SELECT count(*) from job group by status WHERE job = 'dirty'
MultiGauge statuses = MultiGauge.builder("statuses")
    .tag("job", "dirty")
    .description("The number of widgets in various statuses")
    .baseUnit("widgets")
    .register(registry);

// run this periodically whenever you re-run your query
statuses.register(
    resultSet.stream()
        .map(result -> Row.of(Tags.of("status", result.getAsString("status")), result.getAsInt("count")))
        .collect(toList())
);
```

### Timers - 计时器

计时器用于测量短时延迟和此类事件的频率。 Timer 的所有实现至少将总时间和事件计数报告为单独的时间序列。虽然您可以将计时器用于其他用例，但请注意，不支持负值，并且记录更长的持续时间可能会导致总时间溢出 Long.MAX_VALUE 纳秒（292.3 年）。

```java
public interface Timer extends Meter {
    ...
    void record(long amount, TimeUnit unit);
    void record(Duration duration);
    double totalTime(TimeUnit unit);
}
```

```java
Timer timer = Timer
    .builder("my.timer")
    .description("a description of what this timer does") // optional
    .tags("region", "test") // optional
    .register(registry);
```

#### Recording Blocks of Code - 统计代码块执行的时间

```java
timer.record(() -> dontCareAboutReturnValue());
timer.recordCallable(() -> returnValue());
Runnable r = timer.wrap(() -> dontCareAboutReturnValue()); // Wrap Runnable or Callable and return the instrumented version of it for use later.
Callable c = timer.wrap(() -> returnValue());
```

> 在您想要测量时间的任何情况下，您都应该使用 Timer 而不是 DistributionSummary。

#### Storing Start State in Timer.Sample - 基于stop的形式统计代码执行的时间

```java
Timer.Sample sample = Timer.start(registry);
// do stuff
Response response = ...
sample.stop(registry.timer("my.timer", "response", response.status()));
```

#### The @Timed annotation

`micrometer-core`模块包含一个`@Timed`注解，框架可以使用该注解为特定类型的方法（例如服务于 Web 请求端点的方法）或更一般地为所有方法添加计时支持。

> `Micrometer`的`Spring Boot`配置无法识别任意方法上的`@Timed`。

此外，`micrometer-core`中还包含一个正在孵化的`AspectJ`切片。您可以通过编译/加载`AspectJ`织入或通过以其他方式（例如`Spring AOP`）解释`AspectJ`切片和代理目标方法的框架实现在您的应用程序中使用它。这是一个示例`Spring AOP`配置：

```java
@Configuration
public class TimedConfiguration {
   @Bean
   public TimedAspect timedAspect(MeterRegistry registry) {
      return new TimedAspect(registry);
   }
}
```

应用`TimedAspect`使`@Timed`可用于`AspectJ`代理实例中的任意方法，如以下示例所示：

```java
@Service
public class ExampleService {

  @Timed
  public void sync() {
    // @Timed will record the execution time of this method,
    // from the start and until it exits normally or exceptionally.
    ...
  }

  @Async
  @Timed
  public CompletableFuture<?> async() {
    // @Timed will record the execution time of this method,
    // from the start and until the returned CompletableFuture
    // completes normally or exceptionally.
    return CompletableFuture.supplyAsync(...);
  }
}
```

#### Function-tracking Timers

`FunctionTimer`有点复杂，感觉用不到。

#### Pause Detection

`Micrometer`使用`LatencyUtils`包来补偿协调遗漏——由系统和`VM`暂停引起的额外延迟，这些延迟会使您的延迟统计数据向下倾斜。

#### Memory Footprint Estimation - Timer的内存占用估计

又是一堆数学公示，直接跳过。。。

### Distribution Summaries - 分布摘要

分布摘要跟踪事件的分布。它在结构上类似于定时器，但记录的是不代表时间单位的值。例如，您可以使用分布摘要来衡量到达服务器的请求的负载大小。

```java
DistributionSummary summary = registry.summary("response.size");
```

```java
DistributionSummary summary = DistributionSummary
    .builder("response.size")
    .description("a description of what this summary does") // optional
    .baseUnit("bytes") // optional 添加基本​​单元以获得最大的便携性。基本单位是某些监控系统命名约定的一部分。如果您忘记了，将其关闭并违反命名约定也不会产生不利影响。
    .tags("region", "test") // optional
    .scale(100) // optional 您可以提供一个比例因子，每个记录的样本在记录时乘以该比例因子。
    .register(registry);
```

#### Scaling and Histograms

Micrometer 预选的百分位数直方图桶都是从 1 到 Long.MAX_VALUE 的整数。目前，minimumExpectedValue 和 maximumExpectedValue 用于控制桶集的基数。如果我们尝试检测您的最小值/最大值产生的范围较小，并将预选的存储桶域扩展到您的摘要范围，我们就没有其他方法来控制存储桶基数。
相反，如果您的摘要域受到更多限制，请按固定系数缩放您的摘要范围。到目前为止，我们听到的最多用例是域为 [0,1] 的比率的汇总。在这种情况下，我们可以使用以下代码创建 0 到 100 之间的值：

```java
DistributionSummary.builder("my.ratio").scale(100).register(registry);
```

这样，比率在 [0,100] 范围内结束，我们可以将 maximumExpectedValue 设置为 100。如果您关心特定比率，可以将其与自定义 SLO 边界配对：

```java
DistributionSummary.builder("my.ratio")
   .scale(100)
   .serviceLevelObjectives(70, 80, 90)
   .register(registry)
```

#### Memory Footprint Estimation - Distribution Summaries的内存占用估计

又是一堆数学公示，直接跳过。。。

### Long Task Timers

长任务计时器是一种特殊类型的计时器，可让您在正在测量的事件仍在运行时测量时间。一个普通的`Timer`只记录任务完成后的持续时间。

长任务计时器至少发布以下统计信息：

- 活动任务数
- 活动任务的总持续时间
- 活动任务的最长持续时间

与常规计时器不同，长任务计时器不会发布有关已完成任务的统计信息。

考虑一个从数据存储中刷新元数据的后台进程。例如，Edda 缓存 AWS 资源，例如实例、卷、自动扩展组等。通常所有数据都可以在几分钟内刷新。如果 AWS 服务有问题，可能需要更长的时间。长任务计时器可用于跟踪刷新元数据的活动时间。

例如，在`Spring`应用程序中，使用`@Scheduled`实现这种长时间运行的进程是很常见的。`Micrometer`提供了一个特殊的`@Timed`注解，用于使用长任务计时器检测这些进程：

```java
@Timed(value = "aws.scrape", longTask = true)
@Scheduled(fixedDelay = 360000)
void scrapeResources() {
    // find instances, volumes, auto-scaling groups, etc...
}
```

使用@Timed 来实现某些事情取决于应用程序框架。如果您选择的框架不支持它，您仍然可以使用长任务计时器：

```java
LongTaskTimer scrapeTimer = registry.more().longTaskTimer("scrape");
void scrapeResources() {
    scrapeTimer.record(() => {
        // find instances, volumes, auto-scaling groups, etc...
    });
}
```

如果我们想在此进程超过阈值时发出警报，使用长任务计时器，我们会在超过阈值后的第一个报告间隔收到该警报。使用常规计时器，我们要等到流程完成后的第一个报告间隔（一个多小时后）才会收到警报！

```java
LongTaskTimer longTaskTimer = LongTaskTimer
    .builder("long.task.timer")
    .description("a description of what this timer does") // optional
    .tags("region", "test") // optional
    .register(registry);
```

### Histograms and Percentiles - 直方图和百分位数

`Timers`和`distribution summaries`支持收集数据以观察它们的百分位分布。查看百分位数的主要方法有两种：

- Percentile histograms（百分位直方图）：`Micrometer`将值累积到底层直方图，并将一组预定的桶发送到监控系统。监控系统的查询语言负责计算此直方图的百分位数。目前，只有`Prometheus`、`Atlas`和`Wavefront`分别通过`histogram_quantile`、`:percentile`和`hs()`支持基于直方图的百分位数近似。
- Client-side percentiles（客户端百分位数）：`Micrometer`计算每个仪表`ID`（一组名称和标签）的百分位近似值，并将百分位值发送到监控系统。这不如百分位直方图灵活，因为不可能跨标签聚合百分位近似值。然而，它为不支持基于直方图的服务器端百分位计算的监控系统提供了对百分位分布的某种程度的洞察。

以下示例构建了一个带有直方图的计时器：

```java
Timer.builder("my.timer")
   .publishPercentiles(0.5, 0.95) // 用于发布在您的应用程序中计算的百分位值。这些值不可跨维度聚合。
   .publishPercentileHistogram() // 表示发布百分位直方图，即上面的1这种实现
   .serviceLevelObjectives(Duration.ofMillis(100)) // 用于发布包含由您的 SLO 定义的存储桶的累积直方图。
   .minimumExpectedValue(Duration.ofMillis(1)) // 控制 publishPercentileHistogram 传送的桶数，并控制底层 HdrHistogram 结构的准确性和内存占用。
   .maximumExpectedValue(Duration.ofSeconds(10))
```

由于将百分位数传送到监控系统会生成额外的时间序列，因此通常最好不要在作为依赖项包含在应用程序中的核心库中配置它们。相反，应用程序可以通过使用仪表过滤器为某些计时器和分发摘要集打开此行为。

例如，假设我们在一个公共库中有一些计时器。我们在这些计时器名称前加上了`myservice`：

```java
registry.timer("myservice.http.requests").record(..);
registry.timer("myservice.db.requests").record(..);
```

我们可以使用仪表过滤器为两个计时器打开客户端百分位数：

```java
registry.config().meterFilter(
    new MeterFilter() {
        @Override
        public DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
            if(id.getName().startsWith("myservice")) {
                return DistributionStatisticConfig.builder()
                    .percentiles(0.95)
                    .build()
                    .merge(config);
            }
            return config;
        }
    });
```

## Spring boot Metrics

> `https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#actuator.metrics`

`Spring Boot Actuator`为`Micrometer`提供依赖管理和自动配置，`Micrometer`是一个支持众多监控系统的应用程序指标门面，包括：

- AppOptics
- Atlas
- Datadog
- Dynatrace
- Elastic
- Ganglia
- Graphite
- Humio
- Influx
- JMX
- KairosDB
- New Relic
- Prometheus
- SignalFx
- Simple (in-memory)
- StatsD
- Wavefront

### Getting started

`Spring Boot`自动配置一个`CompositeMeterRegistry`并为它在类路径上找到的每个受支持的实现添加一个注册表到复合。在运行时类路径中依赖于`micrometer-registry-{system}`足以让`Spring Boot`配置注册表。

大多数注册管理机构都有共同的特点。例如，即使`Micrometer`注册表实现位于类路径中，您也可以禁用特定的注册表。以下示例禁用 Datadog：

```properties
management.datadog.metrics.export.enabled=false
```

您还可以禁用所有注册表，除非注册表特定属性另有说明，如以下示例所示：

```properties
management.defaults.metrics.export.enabled=false
```

Spring Boot 还将任何自动配置的注册表添加到`Metrics`类的全局静态复合注册表中，除非您明确告诉它不要：

```properties
management.metrics.use-global-registry=false
```

您可以注册任意数量的`MeterRegistryCustomizer` beans以进一步配置注册表，例如应用公共标签，然后再向注册表注册任何仪表：

```kotlin
@Configuration(proxyBeanMethods = false)
class MyMeterRegistryConfiguration {

    @Bean
    fun metricsCommonTags(): MeterRegistryCustomizer<MeterRegistry> {
        return MeterRegistryCustomizer { registry ->
            registry.config().commonTags("region", "us-east-1")
        }
    }

}
```

您可以通过更具体地通用类型来将自定义应用到特定的注册表实现：

```kotlin
@Configuration(proxyBeanMethods = false)
class MyMeterRegistryConfiguration {

    @Bean
    fun graphiteMetricsNamingConvention(): MeterRegistryCustomizer<GraphiteMeterRegistry> {
        return MeterRegistryCustomizer { registry: GraphiteMeterRegistry ->
            registry.config().namingConvention(this::name)
        }
    }

    private fun name(name: String, type: Meter.Type, baseUnit: String?): String {
        return  ...
    }

}
```

### Supported Monitoring Systems

临时只需要看Prometheus的部分

#### Prometheus

`Prometheus`期望抓取或轮询单个应用程序实例的指标。`Spring Boot`在`/actuator/prometheus`提供了一个执行器端点，以呈现具有适当格式的`Prometheus`抓取。

> 默认情况下，端点不可用，必须主动公开。

以下示例`scrape_config`添加到`prometheus.yml`：

```yaml
scrape_configs:
  - job_name: "spring"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["HOST:PORT"]
```

对于可能存在时间不足以被抓取的临时或批处理作业，您可以使用`Prometheus Pushgateway`支持将指标公开给`Prometheus`。要启用`Prometheus Pushgateway`支持，请将以下依赖项添加到您的项目中：

```xml
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_pushgateway</artifactId>
</dependency>
```

当类路径上存在`Prometheus Pushgateway`依赖项并且`management.prometheus.metrics.export.pushgateway.enabled`属性设置为`true`时，会自动配置 `PrometheusPushGatewayManager` bean。这个`manages`将指标推送到`Prometheus Pushgateway`。
您可以使用`management.prometheus.metrics.export.pushgateway`下的属性调整`PrometheusPushGatewayManager`。对于高级配置，您还可以提供自己的 `PrometheusPushGatewayManager` bean。

### Supported Metrics and Meters

`Spring Boot`为各种技术提供自动仪表注册。在大多数情况下，默认值提供了可以发布到任何支持的监控系统的合理指标。

#### JVM Metrics

自动配置通过使用核心`Micrometer`类启用`JVM`指标。`JVM`指标在`jvm` meter name下发布，主要提供了如下：

- 各种内存和缓冲池细节：
- 与垃圾回收相关的统计数据
- 线程利用率：
- 加载和卸载的类数
- JVM 版本信息
- JIT编译时间

其中promethues暴露的具体指标为：

- 可用于内存管理的最大内存量（以字节为单位）： jvm_memory_max_bytes gauge
- 池中缓冲区数量的估计：jvm_buffer_count_buffers gauge
- Java 虚拟机为此缓冲池使用的内存的估计值：jvm_buffer_memory_used_bytes gauge
- 提交给 Java 虚拟机使用的内存量（以字节为单位）：jvm_memory_committed_bytes gauge
- 长寿命堆内存池的最大大小：jvm_gc_max_data_size_bytes gauge
- 在一次 GC 之后到下一次之前增加（年轻的）堆内存池的大小：jvm_gc_memory_allocated_bytes_total counter
- GC 前到 GC 后老年代内存池大小的正增长计数：jvm_gc_memory_promoted_bytes_total counter
- GC 暂停花费的时间：jvm_gc_pause_seconds summary
- GC 暂停花费的时间：jvm_gc_pause_seconds_max gauge
- 当前活动守护线程数： jvm_threads_daemon_threads gauge
- 当前活动线程数，包括守护线程和非守护线程： jvm_threads_live_threads gauge
- 自 Java 虚拟机开始执行以来卸载的类总数：jvm_classes_unloaded_classes_total counter
- Java 虚拟机中当前加载的类数：jvm_classes_loaded_classes gauge
- 回收后长寿命堆内存池的大小：jvm_gc_live_data_size_bytes gauge
- 该池中缓冲区总容量的估计：jvm_buffer_total_capacity_bytes gauge
- 已用内存量：jvm_memory_used_bytes gauge
- 当前具有 NEW 状态的线程数：jvm_threads_states_threads gauge
- 自 Java 虚拟机启动或峰值重置以来的峰值活动线程数：jvm_threads_peak_threads gauge

#### System Metrics

自动配置通过使用核心`Micrometer`类启用系统指标。系统指标在`system`,`process`,`disk` meter name下发布，主要提供了如下：

- CPU指标
- 文件描述符指标
- 正常运行时间指标（应用程序运行的时间量和绝对开始时间的固定标准）
- 可用磁盘空间

其中promethues暴露的具体指标为：

- JVM的正常运行时间：process_uptime_seconds gauge
- JVM可用的处理器数量：system_cpu_count gauge
- 自 unix 纪元以来进程的开始时间：process_start_time_seconds gauge
- 整个系统的“最近的 CPU 使用率”：system_cpu_usage gauge
- 最大文件描述符数：process_files_max_files gauge
- Java 虚拟机进程的“最近的 CPU 使用率”：process_cpu_usage gauge
- 打开的文件描述符计数：process_files_open_files gauge

#### Application Startup Metrics

自动配置公开应用程序启动时间指标：

- `application.started.time`：启动应用程序所用的时间。
- `application.ready.time`：应用程序准备好服务请求所花费的时间。

指标由应用程序类的完全限定名称标记。

#### Logger Metrics

自动配置为`Logback`和`Log4J2`启用事件指标。详细信息发布在`og4j2.events`或`logback.events` meter name下。

其中promethues暴露的具体指标为：

- 写入日志的错误级别事件数：logback_events_total counter

#### Task Execution and Scheduling Metrics

只要底层`ThreadPoolExecutor`可用，自动配置就可以检测所有可用的`ThreadPoolTask​​Executor`和`ThreadPoolTask​​Scheduler` `bean`。指标由`executor`的名称标记，该名称派生自 `bean`名称。

#### Spring MVC Metrics

自动配置支持对由`Spring MVC`控制器和功能处理程序处理的所有请求进行检测。默认情况下，生成的指标名称为`http.server.requests`。您可以通过设置`management.observations.http.server.requests.name`属性来自定义名称。

By default, the following KeyValues are created:

Name|Description
---|---
exception (required)|Name of the exception thrown during the exchange, or KeyValue#NONE_VALUE} if no exception happened.
method (required)|Name of HTTP request method or "none" if the request was not received properly.
outcome (required)|Outcome of the HTTP server exchange.
status (required)|HTTP response raw status code, or "UNKNOWN" if no response was created.
uri (required)|URI pattern for the matching handler if available, falling back to REDIRECTION for 3xx responses, NOT_FOUND for 404 responses, root for requests with no path info, and UNKNOWN for all other requests.

详见：`https://docs.spring.io/spring-framework/docs/6.0.4/reference/html/integration.html#integration.observability.http-server`

To add to the default tags, provide a @Bean that extends DefaultServerRequestObservationConvention from the org.springframework.http.server.observation package. To replace the default tags, provide a @Bean that implements ServerRequestObservationConvention.

默认情况下，处理所有请求。要自定义过滤器，请提供一个实现`FilterRegistrationBean<WebMvcMetricsFilter>`的`@Bean`。

其中promethues暴露的具体指标为：

- http_server_requests_seconds summary / count

#### Spring WebFlux Metrics

暂时用不到

#### Jersey Server Metrics

暂时用不到

#### HTTP Client Metrics

`Spring Boot Actuator`管理`RestTemplate`和`WebClient`的检测。暂时用不到

#### Tomcat Metrics

仅当启用`MBeanRegistry`时，自动配置才启用`Tomcat`的检测。默认情况下，`MBeanRegistry`是禁用的，但您可以通过将`server.tomcat.mbeanregistry.enabled`设置为`true`来启用它。`Tomcat`指标在`tomcat` meter name下发布。

其中promethues暴露的具体指标为：

- tomcat_sessions_rejected_sessions_total counter
- tomcat_sessions_alive_max_seconds gauge
- tomcat_sessions_active_current_sessions gauge
- tomcat_sessions_created_sessions_total counter
- tomcat_sessions_active_max_sessions gauge
- tomcat_sessions_expired_sessions_total counter

#### Cache Metrics

自动配置允许在启动时检测所有可用的缓存实例，指标以`cache`为前缀。缓存检测针对一组基本指标进行了标准化。此外，还提供了特定于缓存的指标。
支持以下缓存库：

- Cache2k
- Caffeine
- Hazelcast
- Any compliant JCache (JSR-107) implementation
- Redis

只有在启动时配置的缓存才会绑定到注册表。对于未在缓存配置中定义的缓存，例如在运行阶段或在启动阶段后以编程方式创建的缓存，需要显式注册。`CacheMetricsRegistrar` `bean`可用于简化该过程。

#### Spring GraphQL Metrics

暂时用不到

#### DataSource Metrics

自动配置启用了所有可用数据源对象的检测，这些对象具有以`jdbc.connections`为前缀的指标。数据源检测会生成表示池中当前活动、空闲、允许的最大连接数和允许的最小连接数的仪表。

> 默认情况下，`Spring Boot`为所有支持的数据源提供元数据。如果不支持您喜欢的数据源，您可以添加额外的`DataSourcePoolMetadataProvider` `beans`。有关示例，请参阅 `DataSourcePoolMetadataProvidersConfiguration`。

此外，特定于`Hikari`的指标以`hikaricp`前缀公开。每个指标都由池的名称标记（您可以使用`spring.datasource.name`来控制它）。

#### Hibernate Metrics

#### Spring Data Repository Metrics

#### RabbitMQ Metrics

自动配置允许使用名为`rabbitmq`的指标检测所有可用的`RabbitMQ`连接工厂。

其中promethues暴露的具体指标为：

- rabbitmq_rejected_total counter
- rabbitmq_failed_to_publish_total counter
- rabbitmq_acknowledged_total counter
- rabbitmq_published_total counter
- rabbitmq_not_acknowledged_published_total counter
- rabbitmq_consumed_total counter
- rabbitmq_connections gauge
- rabbitmq_unrouted_published_total counter
- rabbitmq_channels gauge
- rabbitmq_acknowledged_published_total counter

#### Spring Integration Metrics

#### Kafka Metrics

#### MongoDB Metrics

#### Jetty Metrics

#### @Timed Annotation Support

#### Redis Metrics

自动配置为自动配置的`LettuceConnectionFactory`注册一个`MicrometerCommandLatencyRecorder`。有关更多详细信息，请参阅 [Lettuce 文档](https://lettuce.io/core/6.2.2.RELEASE/reference/index.html#command.latency.metrics.micrometer)。

### Registering Custom Metrics

要注册自定义指标，请将`MeterRegistry`注入您的组件：

```kotlin
@Component
class MyBean(registry: MeterRegistry) {

    private val dictionary: Dictionary

    init {
        dictionary = Dictionary.load()
        registry.gauge("dictionary.size", Tags.empty(), dictionary.words.size)
    }

}
```

如果您的指标依赖于其他`beans`，我们建议您使用`MeterBinder`来注册它们：

```kotlin
class MyMeterBinderConfiguration {

    @Bean
    fun queueSize(queue: Queue): MeterBinder {
        return MeterBinder { registry ->
            Gauge.builder("queueSize", queue::size).register(registry)
        }
    }

}
```

使用`MeterBinder`可确保设置正确的依赖关系，并确保在检索指标值时`bean`可用。如果您发现您重复检测一组跨组件或应用程序的指标，则`MeterBinder`实现也很有用。

> 默认情况下，来自所有`MeterBinder` `bean`的指标会自动绑定到`Spring`管理的`MeterRegistry`。

### Customizing Individual Metrics

如果您需要将自定义应用到特定的`Meter`实例，您可以使用`io.micrometer.core.instrument.config.MeterFilter`接口。

例如，如果要将所有以`com.example`开头的`meter IDs`的`mytag.region`标签重命名为`mytag.area`，您可以执行以下操作：

```kotlin
@Configuration(proxyBeanMethods = false)
class MyMetricsFilterConfiguration {

    @Bean
    fun renameRegionTagMeterFilter(): MeterFilter {
        return MeterFilter.renameTag("com.example", "mytag.region", "mytag.area")
    }

}
```

默认情况下，所有`MeterFilter bean`都自动绑定到`Spring`管理的`MeterRegistry`。确保使用`Spring`管理的`MeterRegistry`而不是`Metrics`上的任何静态方法来注册您的指标。这些静态方法使用非`Spring`管理的全局注册表。

#### Common Tags

通用标签一般用于对运行环境进行`dimensional drill-down`，如`host`、`instance`、`region`、`stack`等。 `Commons tags`应用于所有仪表并且可以配置，如以下示例所示：

```properties
management.metrics.tags.region=us-east-1
management.metrics.tags.stack=prod
```

前面的示例将`region`和`stack`标签和值为`us-east-1`和`prod`添加到了所有的仪表。

> 如果您使用`Graphite`，公共标签的顺序很重要。由于使用这种方法无法保证公共标签的顺序，因此建议`Graphite`用户使用一个自定义的`MeterFilter`来实现该功能。

#### Per-meter Properties

In addition to `MeterFilter beans`, you can apply a limited set of customization on a per-meter basis using properties. Per-meter customizations are applied, using Spring Boot’s `PropertiesMeterFilter`, to any meter IDs that start with the given name. The following example filters out any meters that have an ID starting with `example.remote`.
