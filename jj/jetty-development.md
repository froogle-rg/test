<table>
  <tr>
    <td><a href="oauth-client">&lt; 10. OAuth Enabled SDKs with Google OAuth2 Client</a></td>
    <td align="right"><a href="span-trace-id">12. Span/Trace IDs &gt;</a></td>
  </tr>
</table>
# 11. Embedded Development Server with Jetty

To aid local development and testing, an embedded [Jetty](http://www.eclipse.org/jetty/) server is bundled with the webapp module in such a way that a simple Maven command can wire up the entire application and serve it up using Jetty as the servlet container.

You can start your service with Jetty using the following commands:

```shell
cd webapp
mvn jetty:run
```

See the [jetty:run](http://www.eclipse.org/jetty/documentation/current/jetty-maven-plugin.html#jetty-run-goal) goal for more details.

Of course, nothing is stopping you from deploying the application war file into a servlet container of your choice.  Jetty is simply provided as a convenience to the developer as well as accelerating ramp-up time of new resources.

## Configuration

The test server is configured using the jetty-maven-plugin within the webapp ```pom.xml``` as follows:

```xml
  <plugin>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-maven-plugin</artifactId>
    <configuration>
       <jettyXml>${basedir}/src/test/jetty/jetty-jmx.xml</jettyXml>
       <testClassesDirectory>${basedir}/target/test-classes</testClassesDirectory>
       <useTestScope>true</useTestScope>
       <systemProperties>
         <systemProperty>
           <name>spring.profiles.active</name>
           <value>dev</value>
         </systemProperty>
       </systemProperties>
    </configuration>
  </plugin>
```

Note that the Spring profile ```dev``` must be activated in order for the Spring context to load up local environment properties and beans during Jetty startup.  This profile activates certain categories of Spring configurations used for development purposes.  See the following snippet in ```spring-bootstrap.xml```:

```xml
  <beans profile="dev">
    <context:property-placeholder location="classpath:/app.conf" />
  </beans>
```

This tells the Spring context that when the profile is activated via ```-Dspring.profiles.active=dev``` to go ahead and load properties from a classpath resource named ```app.conf```.  If this profile is _not_ activated (for example, in stage or production environments) then app.conf is not used and the properties are loaded from another [external] source.

The bundled Jetty server is automatically included for you when using the [Maven archetype](quickstart) to initialize new projects.  This option __must__ be included in every service to promote homogeneity, do not remove it.

<table>
  <tr>
    <td><a href="oauth-client">&lt; 10. OAuth Enabled SDKs with Google OAuth2 Client</a></td>
    <td align="right"><a href="span-trace-id">12. Span/Trace IDs &gt;</a></td>
  </tr>
</table>