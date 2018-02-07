<table>
  <tr>
    <td><a href="proto-readers-writers">&lt; 1.3. Protobuf Reader/Writer</a></td>
    <td align="right"><a href="implementation-server">2. Server &gt;</a></td>
  </tr>
</table>
# 1.4. Core Configuration

The ```core``` module contains readers and writers for all resources in the service.  The framework needs to be made aware of these objects and so will need to have them wired up.  On the server side, this is done via Spring configuration within the ```spring-view.xml``` file.  Here is an example of setting up the entity readers:

```xml
...

<!-- Create the JSON reader -->
<bean id="bookJsonReader" class="com.covisint.platform.book.core.book.io.json.BookJsonReader" />
<!-- Create the protobuf reader -->
<bean id="bookProtoReader" class="com.covisint.platform.book.core.book.io.protobuf.BookProtobufReader" />
<!-- Create all other readers... -->

...

<util:list id="entityReaders">
   <bean class="com.covisint.core.http.service.core.io.jsonp.HttpServiceErrorReader" />
   <ref bean="bookJsonReader" />
   <ref bean="bookProtoReader" />
   <!-- Inject the rest. -->
</util:list>

...
```

Here we are creating an instance of the book JSON reader, an instance of the book protobuf reader, and then injecting them into a list with id ```entityReaders```.  This is the collection of readers the framework will use during content negotiation.  It will cycle through them all until it finds one to process the resource.  Note that the first item in this list must be the ```HttpServiceErrorReader``` which is used to process any errors that occur.

Next we will show how writers are wired up:

```xml
...

<!-- Create the JSON writer -->
<bean id="bookJsonWriter" class="com.covisint.platform.book.core.book.io.json.BookJsonWriter" />
<!-- Create the protobuf writer -->
<bean id="bookProtoWriter" class="com.covisint.platform.book.core.book.io.protobuf.BookProtobufWriter" />
<!-- Create all other writers... -->

...

<util:list id="entityWriters">
   <bean class="com.covisint.core.http.service.core.io.jsonp.HttpServiceErrorWriter" />
   <ref bean="bookJsonWriter" />
   <ref bean="bookProtoWriter" />
   <!-- Inject the rest. -->
</util:list>

...
```

_Note that there will be times where you use two separate readers to read different representations of the same resource.  This is a perfectly legitimate use case, for example you may have ```application/vnd.com.covisint.platform.book.v1+json``` and also ```application/vnd.com.covisint.platform.book.v2+json```.  In this case where there are separate readers exist to handle the two versions, you will have to simply inject both reader classes and name them appropriately, such as ```BookJsonV1Reader``` and ```BookJsonV2Reader```._

<table>
  <tr>
    <td><a href="proto-readers-writers">&lt; 1.3. Protobuf Reader/Writer</a></td>
    <td align="right"><a href="implementation-server">2. Server &gt;</a></td>
  </tr>
</table>