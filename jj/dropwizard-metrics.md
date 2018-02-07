<table>
  <tr>
    <td><a href="guava">&lt; 8. Client-side Caching with Guava</a></td>
    <td align="right"><a href="oauth-client">10. OAuth Enabled SDKs with Google OAuth2 Client &gt;</a></td>
  </tr>
</table>
# 9. Server Metrics with Dropwizard

The framework employs Dropwizard's [Metrics](https://dropwizard.github.io/metrics/3.1.0/) library to inject runtime meters around the core operations that measure and report the following statistics:

* Meters: the rate at which something is being executed, including 1, 5 and 15 minute moving averages.
* Exception Meters: the rate at which something results in an exception, including 1, 5 and 15 minute moving averages.
* Timers: the duration that something takes to execute, along with its distribution.
* Histograms: the statistical distribution of values, including some typical percentiles.

These instruments are placed around the CRUD operations of any ```com.covisint.core.http.service.server.service.BaseResourceService```.  This includes the following methods:

* ```#count```: Method which counts the number of resources of a particular type.
* ```#exists```: Method which checks if a resource exists for a given id.
* ```#multiGet```: Method which performs a parallel fetch of one or more resources at once.
* ```#search```: Method which executes a resource search query.
* ```#update```: Method which updates a resource.
* ```#add```: Method which creates a new resource.
* ```#delete```: Methods which deletes an existing resource.

***
__Extension Point__
    
In addition to the instrumented operations above, any other public instance method may also be instrumented by adding a metrics annotation to it:

* ```@com.codahale.metrics.annotation.Metered```
* ```@com.codahale.metrics.annotation.ExceptionMetered```
* ```@com.codahale.metrics.annotation.Timed```
* ```@com.codahale.metrics.annotation.Gauge```
* ```@com.codahale.metrics.annotation.Counter```

See [this page](https://dropwizard.github.io/metrics/3.1.0/getting-started/) for more details on these annotations.
***

Collecting metrics is only half the story; the library also defines reporters that can export them to various destinations.  The framework uses two by default:

1. ```com.codahale.metrics.Slf4jReporter```: Writes all metrics data out to SLF4J
2. ```com.codahale.metrics.GraphiteReporter```: Writer all metrics data out to a [Graphite](http://graphite.wikidot.com/) instance.

For simple, day-to-day analysis the SLF4J reporter provides a simple way to see how any particular API is being used and performing.  Above and beyond that, Graphite provides a rich graphical dashboard that can be configured to plot these metrics on an ongoing time-series scale.  See the QA Graphite dashboard here for some examples: http://graphite.covisintrnd.com:8080/dashboard/

## Configuration

All configuration is provided in ```spring-metrics.xml```.  This file is specified in the application's ```web.xml``` and uses an adaptor (```com.ryantenney.metrics:metrics-spring:3.1.0```) to allow the Spring context to recognize the metrics annotations and associate them with the central metrics registry set up in the configuration.

The SLF4J reporter is configured like this:

```xml
<metrics:reporter type="slf4j" metric-registry="metrics" period="${slf4j_log_interval:30m}" />
```

You can override the default reporting interval of 30 minutes by specifying your own value in ```slf4j_log_interval```, using suffix __s__ for seconds, __m__ for minutes, __h__ for hours and so on.

The graphite report is configured like this:

```xml
  <metrics:reporter type="graphite" metric-registry="metrics" period="${graphite_log_interval:15s}"
            host="${graphite_host:graphite.covisintrnd.com}" port="${graphite_port:2004}" 
            transport="pickle" batch-size="${graphite_batch_size:100}" />
```

This reporter is configured to use the "pickle" transport which batches up metrics in ```graphite_batch_size``` batches before sending them to the server located at host ```graphite_host``` and port ```graphite_port```.  Here, too, you can specify your own interval at which metrics will be sent to the server, to ensure metrics get exported no less frequently than a given duration.

The metrics configuration is automatically included for you when using the [Maven archetype](quickstart) to initialize new projects.  SLF4J is always configured by default.  Graphite reporting is provided but commented out.  You may uncomment it if needed.  The ```spring-metrics.xml``` __must__ be included in every service to promote homogeneity, do not remove it.

<table>
  <tr>
    <td><a href="guava">&lt; 8. Client-side Caching with Guava</a></td>
    <td align="right"><a href="oauth-client">10. OAuth Enabled SDKs with Google OAuth2 Client &gt;</a></td>
  </tr>
</table>