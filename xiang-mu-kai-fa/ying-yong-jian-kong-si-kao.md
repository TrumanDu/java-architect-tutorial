# 应用监控思考

对于java应用监控，通常我们需要考虑三个方面问题：

* 系统metric监控
* 日志信息监控
* 业务指标监控

接下来我将对这三个方面进行解释，这个是我自己的思考，一定存在很多局限性，欢迎找我交流。

### 系统metric监控

这里系统metic监控主要指的jvm 指标信息，当然如果你使用spring 框架，还会有更多的信息。暂且将类需求都归属于系统metric中。目前主流的做法：开启jmx端口，收集jmx信息。如果是spring,可以使用actualor。

### 日志信息监控

服务出现问题，首先会按照我们的代码设计产生不同级别的日志，我们需要将自己关心的数据能够自动推送给我们。最简单的办法是配置log发邮件的功能。一个是我们能够收到日志的告警，一个是能便捷高效的查看错误日志。

我接触过的方案有查看容器log,或者使用filbeat采集应用的log,借助ELK，实现日志检索的能力。

### 业务指标监控

业务指标更多的是关注的业务性能，比如api的平均响应延迟，吞吐量，数据库响应延迟等不同维度的监控数据。

对于这里的需求，我会更多的借助于第三方框架，例如：elastic APM、Prometheus、SkyWalking

### 产线案例

Spring Boot应用监控有很多方案，例如elastic APM,Prometheus等。各有特色，本次实践采用方案：[Micrometer](https://micrometer.io/docs)+[Prometheus](https://prometheus.io/)+[Grafana](https://grafana.com/)。

选择Micrometer最重要的原因是他的设计很灵活，并且和spring boot 2.x集成度很高。对于jvm的监控很容易集成，难度很小。本次实践包含jvm监控和业务性能指标监控。

#### 环境准备

1. 搭建promethues

   ```text
   docker run \
       -p 9090:9090 \
       --name prometheus
       -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml \
       prom/prometheus
   ```

   ```text
   global:
     scrape_interval:     15s # By default, scrape targets every 15 seconds.
     evaluation_interval: 15s # By default, scrape targets every 15 seconds.
     # scrape_timeout is set to the global default (10s).
   # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
   rule_files:
     # - "first.rules"
     # - "second.rules"

   # A scrape configuration containing exactly one endpoint to scrape:
   # Here it's Prometheus itself.
   scrape_configs:
     # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
     - job_name: 'demo_platform'

       # Override the global default and scrape targets from this job every 5 seconds.
       scrape_interval: 5s

       metrics_path: '/actuator/prometheus'
       # scheme defaults to 'http'.

       static_configs:
         - targets: ['127.0.0.1:8080']
   ```

2. 搭建grafana

   ```text
   docker run -d -p 3000:3000 --name grafana grafana/grafana:6.5.0
   ```

#### Micrometer简介

![](https://micrometer.io/static/media/logo-no-title.d91d5ac6.svg)

Micrometer（译：千分尺） Micrometer provides a simple facade over the instrumentation clients for the most popular monitoring systems. 翻译过来大概就它提供一个门面，类似SLF4j。支持将数据写入到很多监控系统，不过我谷歌下来，很多都是后端接入的是Prometheus.

Micrometer提供了与供应商无关的接口，包括 **timers（计时器）**， **gauges（量规）**， **counters（计数器）**， **distribution summaries（分布式摘要）**， **long task timers（长任务定时器）**。它具有维度数据模型，当与维度监视系统结合使用时，可以高效地访问特定的命名度量，并能够跨维度深入研究。

支持的监控系统：AppOptics ， Azure Monitor ， Netflix Atlas ， CloudWatch ， Datadog ， Dynatrace ， Elastic ， Ganglia ， Graphite ， Humio ， Influx/Telegraf ， JMX ， KairosDB ， New Relic ， Prometheus ， SignalFx ， Google Stackdriver ， StatsD ， Wavefront

#### Micrometer提供的度量类库

`Meter`是指一组用于收集应用中的度量数据的接口，Meter单词可以翻译为"米"或者"千分尺"，但是显然听起来都不是很合理，因此下文直接叫Meter，理解它为度量接口即可。`Meter`是由`MeterRegistry`创建和保存的，可以理解`MeterRegistry`是`Meter`的工厂和缓存中心，一般而言每个JVM应用在使用Micrometer的时候必须创建一个`MeterRegistry`的具体实现。Micrometer中，`Meter`的具体类型包括：`Timer`，`Counter`，`Gauge`，`DistributionSummary`，`LongTaskTimer`，`FunctionCounter`，`FunctionTimer`和`TimeGauge`。一个`Meter`具体类型需要通过名字和`Tag`\(这里指的是Micrometer提供的Tag接口\)作为它的唯一标识，这样做的好处是可以使用名字进行标记，通过不同的`Tag`去区分多种维度进行数据统计。

#### Spring Boot集成

与spring boot 集成，这里的metric主要是由spring actuator 提供

#### 安装

```text
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

#### 配置

```text
management:
  endpoint:
    health:
      enabled: false
  endpoints:
    web:
      exposure:
        include: '*'
        exclude: env,beans
  metrics:
    enable:
      http: false
      hikaricp: false
```

这里有几个注意的点`management.endpoint.health.enabled`只是为了禁用spring 默认的健康检查，非必须。`exclude: env,beans`也不需要配置，只是在我项目中为了减少导出的metric。同理`management.metrics.enable`也是为了减少收集的数据，使用方法为你定义指标的前缀。

只有`management.endpoints.web.exposure.include`为必须的，这里也只是为了导出`/actuator/prometheus`，通过该地址可以访问到响应的metric信息。

#### 可视化

访问 `http://localhost:8080/actuator/prometheus` 即可看到响应的metric信息。

在grafana中中导入[JVM \(Micrometer\)](https://grafana.com/grafana/dashboards/4701)

即可看到如下效果：

![](http://blogstatic.aibibang.com/dashboards.png)

#### 自定义业务性能监控

因为系统遗留监控代码的原因，这里采用的是全局静态方法实现。

```text
    protected static Iterable<Tag> tags(String service, String category, String method) {
        return Tags.of("service", service, "category", category, "method", method);
    }

    protected static Iterable<Tag> tags(String service, String category) {
        return Tags.of("service", service, "category", category);
    }

    public static void controllerMetric(String service, MonitorMetric.MonitorOperationType type, String method, long time) {
        try {
            Metrics.counter(Constants.HTTP_REQUESTS_TOTAL, tags(service, type.name(), method)).increment();
            Metrics.timer(Constants.REQUESTS_LATENCY, tags(service, type.name())).record(Duration.ofMillis(time));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

解释一下，这里可以统计出请求数和请求延迟。

对于每秒请求数据量，可以使用`increase(http_requests_total{job=~"$job",instance=~"$instance"}[1m])`

对于平均请求延迟，可以使用`rate(timer_sum[1m])/rate(timer_count[1m])`

对于Throughput 可以使用`rate(timer_count[1m])`

![](http://blogstatic.aibibang.com/syncbigdata_monitorpng.png)

#### 使用中的困惑

#### 问题

**Percentile histograms**与**Distribution summaries**性能损失还无法确定，不过查看`PrometheusTimer`，结合测试，还是有一定的性能损失，不过这里未深入研究。

#### 全局使用一些开发建议

可以在定义静态方法类，初始化的时候做一点配置，registry可以使用spring 注入进来例如：

```text
@Autowired 
MeterRegistry registry;
```

```text
public MonitorMetric(MeterRegistry registry) {
        registry.config().meterFilter(
                new MeterFilter() {
                    @Override
                    public DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
             if (id.getName().startsWith("requests_latency")) {
                            return DistributionStatisticConfig.builder()
                                    .percentiles(0.5, 0.75, 0.9)
                                    .sla(1)
                                    .expiry(Duration.ofMinutes(1))
                                    .minimumExpectedValue(1L)
                                    .build()
                                    .merge(config);
                        }
                        return config;
                    }
                });
        Metrics.addRegistry(registry)；
    }
```

#### 参考

1. [使用 Micrometer 记录 Java 应用性能指标](https://www.ibm.com/developerworks/cn/java/j-using-micrometer-to-record-java-metric/index.html)
2. [Micrometer 快速入门](https://www.cnblogs.com/cjsblog/p/11556029.html)
3. [JVM应用度量框架Micrometer实战](http://www.throwable.club/2018/11/17/jvm-micrometer-prometheus/)
4. [Micrometer Prometheus](https://micrometer.io/docs/registry/prometheus)

