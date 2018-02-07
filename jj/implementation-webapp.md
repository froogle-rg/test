<table>
  <tr>
    <td><a href="riak-indexer">&lt; 2.6. Riak Indexer</a></td>
    <td align="right"><a href="implementation-client">4. Client &gt;</a></td>
  </tr>
</table>
# 3. Webapp

Unlike the other Maven modules in your service project, the ```webapp``` module doesn't contain any source code.  Its purpose is simply to bundle a deployable war file along with all required Spring configuration.  Its main dependency is the project's ```server``` module, along with any other runtime Maven dependencies that are required to be included.

Inside ```src/main/resources``` you will include all Spring configuration XML files - 

1. ```spring-bootstrap.xml```: Bootstraps the application, mainly just sets up the Spring property placeholder strategy to be used.  For example - 

```xml
    <beans profile="dev">
        <context:property-placeholder location="classpath:/app.conf" />
    </beans>
    <beans profile="!dev">
        <context:property-placeholder />
    </beans>
```

This directs Spring to look for a file named ```app.conf``` at the root of the classpath if the ```dev``` Spring profile is activated.  Otherwise, it will look for all properties in a series of default locations, one of which are JVM args.  The ```dev``` profile is activated in development (local) environments.  All other profiles are targeted for environments such as CloudFoundry where the configuration values are externalized and supplied by the Java runtime.

2. ```spring-view.xml```: This file configured everything in the application related to how Spring does view resolution, content negotiation, and message conversion.  In this file you will configure your entity readers and writers, inject them into a collection so they can be used by the framework, and also set up the entity validators.

3. ```spring-service.xml```: This is used to configure all service beans (anything extending ```com.covisint.core.http.service.server.service.BaseResourceService```) along with their dependencies.  

4. ```spring-persistence.xml```: This file contains all definition of the Riak DAO objects.  It is also used to define the Riak client factory - 

```xml
    <!--  Riak Client -->
    <bean id="riakClient" factory-bean="riakClientFactory" factory-method="create" destroy-method="shutdown"/>

    <bean id="riakClientFactory" class="com.covisint.core.riak.support.client.RiakClientFactory">
        <property name="addresses">
            <list>
                <value>${riak_address}</value>
            </list>
        </property>
    </bean>
```

```riak_address``` is the host used to front the Riak cluster; typically a load-balanced URL.

5. ```spring-cache.xml```: Used to define the cache settings and wire up the Redis and Jedis classes, if so desired.  If this class is left out and not included into the Spring context, then no caching will be done and it is disabled.

6. ```spring-metrics.xml```: Used to configure how the dropwizard metrics component delivers metrics.  By default (if using the Maven Archetype to auto-generate a skeleton project) you will get slf4j logging of metrics data.  However it is easily possible to have the component write out stats to Graphite as well.

All of these Spring configuration files must be specified for inclusion in the Spring context.  This is done via the ```web.xml``` which is located in the module's ```src/main/webapp/WEB-INF/``` directory -

```xml
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
            classpath*:/spring-bootstrap.xml
            classpath*:/spring-persistence.xml
            classpath*:/spring-service.xml
            classpath*:/spring-cache.xml
            classpath*:/spring-metrics.xml
            classpath*:/spring-view.xml
        </param-value>
    </context-param>
```

<table>
  <tr>
    <td><a href="riak-indexer">&lt; 2.6. Riak Indexer</a></td>
    <td align="right"><a href="implementation-client">4. Client &gt;</a></td>
  </tr>
</table>