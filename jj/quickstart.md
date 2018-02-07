<table>
  <tr>
    <td><a href="principles">&lt; II ReSTful Principles</a></td>
    <td align="right"><a href="implementation-basics">IV Implementation: The Basics &gt;</a></td>
  </tr>
</table>
# III Quick Start: Using the Maven Archetype

A Maven archetype is available to enable developers to quickly spin up a properly-structured HTTP service based on this framework.  After using this archetype to generate a new project, the developer will be able to immediately start contributing project-specific code.  The new project will be a Maven multi-module project complete with checkstyle configuration and all eclipse project settings included.

The archetype jar is available in Nexus, but in case you want to download it locally here are the Maven coordinates:

```xml
  <dependency>
    <groupId>com.covisint.core.maven.archetype</groupId>
    <artifactId>http-service-archetype</artifactId>
    <version>2.0.0.RELEASE</version>
  </dependency>
```

Based on this particular archetype version, you can start project generation with the following command:

```
mvn archetype:generate -DarchetypeGroupId=com.covisint.core.maven.archetype 
    -DarchetypeArtifactId=http-service-archetype -DarchetypeVersion=2.0.0.RELEASE
```

This Maven goal will prompt you for some input parameters used for your new project:

```
Define value for property 'groupId': : com.covisint.service.peanut  
Define value for property 'artifactId': : peanut-service
Define value for property 'version': : TRUNK-SNAPSHOT
Define value for property 'package': : com.covisint.service.peanut
Define value for property 'httpServiceId': : peanut
```

In this case, we are creating ```com.covisint.service.peanut:peanut-service:TRUNK-SNAPSHOT``` and must define the custom property ```httpServiceId```.  This is used for many aspects throughout the project configuration.  The rule of thumb is to assign this property the singular noun of your resource.  

For example, some _valid_ httpServiceId values might be:

* peanut
* book
* device
* person

Some _invalid_ examples are:

* peanuts
* people
* organizations 

Finally the Maven plugin will prompt you to confirm, after which your project is created.  It is always good practice to build and run the empty project just to make sure it is set up properly:

```shell
cd peanuts
mvn package
cd webapp
mvn jetty:run
```

If all goes well, the above statements will build your project and start the service on a local Jetty instance on http://localhost:8080/peanuts.

<table>
  <tr>
    <td><a href="principles">&lt; II ReSTful Principles</a></td>
    <td align="right"><a href="implementation-basics">IV Implementation: The Basics &gt;</a></td>
  </tr>
</table>