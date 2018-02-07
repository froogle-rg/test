<table>
  <tr>
    <td><a href="quickstart">&lt; III Quick Start: Using the Maven Archetype</a></td>
    <td align="right"><a href="implementation-core">1. Core &gt;</a></td>
  </tr>
</table>
## IV Implementation: The Basics

This section will take you through the steps of creating your first resource-based APIs.  It assumes you have generated an initial project via the [Maven archetype](quickstart) and that the project builds and starts up via executing ```mvn jetty:run``` from within the webapp directory.

There is a natural bottom-up order of development that will take place.  First, we will create the resource class(es).  Then we create the readers and writers that will marshal and unmarshal those resources.  Next we create the server layers - the controller, service and DAO layers.  Then we add validation routines and finally we write the client classes.

<table>
  <tr>
    <td><a href="quickstart">&lt; III Quick Start: Using the Maven Archetype</a></td>
    <td align="right"><a href="implementation-core">1. Core &gt;</a></td>
  </tr>
</table>