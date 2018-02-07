<table>
  <tr>
    <td><a href="jetty-development">&lt; 11. Embedded Development Server with Jetty</a></td>
    <td align="right"><a href="errors">13. Errors &gt;</a></td>
  </tr>
</table>
# 12. Span/Trace IDs

There is a servlet request filter - ```com.covisint.core.trace.servlet.SpanFilter``` - which intercepts all HTTP requests and appends some special values onto the request processing thread.  These values are used to "trace" the request through the system, whether that service is the root of the call hierarchy or several layers down in the tree.  

For example, service A may make downstream calls to services B and C.  The purpose of the span filter is to correlate the request flowing through system A, B and C in a way that it can be viewed as one logical thread.  This is a simple scenario; consider real world systems where the call chain involves dozens of downstream components.

These correlation IDs are passed from _calling_ service to _called_ service via special request headers, and are used to construct the complete call tree when examining logs.  The pattern is modeled closely after Google's work with [Dapper](http://research.google.com/pubs/pub36356.html).

The main point of all this is that we can use SLF4J's [MDC](http://www.slf4j.org/api/org/slf4j/MDC.html) feature to print these thread local values on the log messages.

A sample log entry may look like this:

```
2015-10-05:17-05-27 c.c.c.h.s.c.i.j.AbstractResourceWriter#createWriterFactory:87 - A message. [5083388048999512297|404074600175177042|273721736183621]
```

In this case, the three numbers within the square brackets hold the trace id, span id and parent span id.  If this was the first service being called, and therefore is the root of the call tree, then there is no parent span id and the third value will be '0'.

## Configuration

The span filter is declared in the ```web.xml``` like this:

```xml
  <filter>
    <filter-name>TraceFilter</filter-name>
    <filter-class>com.covisint.core.trace.servlet.SpanFilter</filter-class>
    <init-param>
        <param-name>spanName</param-name>
        <param-value>peanut-service</param-value>
    </init-param>
  </filter>

  <filter-mapping>
    <filter-name>TraceFilter</filter-name>
    <servlet-name>Dispatcher</servlet-name>
  </filter-mapping>
```

The span filter is automatically configured for you when using the [Maven archetype](quickstart) to initialize new projects.  The span filter __must__ be included in every service to promote homogeneity, do not remove it.

<table>
  <tr>
    <td><a href="jetty-development">&lt; 11. Embedded Development Server with Jetty</a></td>
    <td align="right"><a href="errors">13. Errors &gt;</a></td>
  </tr>
</table>