<table>
  <tr>
    <td><a href="json-readers-writers">&lt; 1.2. JSON Reader/Writer</a></td>
    <td align="right"><a href="core-configuration">1.4. Configuration &gt;</a></td>
  </tr>
</table>
# 1.3. Protobuf Reader/Writer

The framework allows for entity readers and writers to use the [Google Protocol Buffers](https://developers.google.com/protocol-buffers/?hl=en) format in the resource representation.  This is a binary data structure that is roughly 85% leaner than JSON and 50% less CPU intensive (although these figures depend greatly on the contents of the data that is used for the comparison).  All HTTP clients should prefer the protobuf media type over the JSON media type since they are used in application-to-application communication and therefore the transmitted data does not need to be in a human-readable format such as JSON.

## Package Naming Convention

The convention for naming protobuf marshaler packages is as follows:

```
com.covisint.platform.{serviceId}.core.{entityOrResourceType}.io.protobuf
```

For example:

```
com.covisint.platform.book.core.book.io.protobuf
```

This package will contain all classes that read and write the book resources to and from protobuf.  Before creating the readers and writers, however, we need to define the protobuf classes that will be used to create the binary data stream.

## Proto Source Files

First and foremost, you will need to define the structure of the protobuf payload with something called an ```IDL``` - an Interface Definition Language.  This is a source file where you set up the schema of the payload, after which you will compile it with the ```protoc``` compiler and it will auto-generate the corresponding Java classes for you.  It is these java classes that make up the protobuf module of your project.  Your protobuf readers and writers will convert these POJOs to and from resources.

Let's break down the steps needed to define, generate and compile proto files:

1. Download the [protoc compiler](https://developers.google.com/protocol-buffers/docs/downloads), install it and place it on your system path.

2. Make sure it is installed and operational by issuing the following command: ```protoc --version```.  You should see a response like __libprotoc 2.6.1__ where 2.6.1 can be replaced with the version you installed.

3. The ```protobuf``` Maven module is where the ```.proto``` file resides as well as the generated source.  Generally there will be one IDL file per resource.  For example, ```peanut.proto``` would contain all protobuf structure definitions for the peanut resource representation.  Here is an example of what this file may look like:

```
import "framework.proto"; // specify framework.proto location

package com.covisint.core.protobuf.message;

option java_package = "com.covisint.platform.peanut.core.protobuf.message";
option java_outer_classname = "PeanutProtos";

// message type representing a peanut
message ProtoPeanut {
  optional ProtoCoreResource core_resource = 1;    // the core resource wrapped by this peanut
  optional string name = 2;     // the peanut's name...they have names, ya know
  optional string shape = 3;    // the peanut shape

  extensions 100 to 199;
}
```

The syntax and conventions for generating correct protobuf IDL files are beyond the scope of this guide.  [Consult the documentation](https://developers.google.com/protocol-buffers/docs/proto?hl=en) to learn more.

4. Put your ```.proto``` files in ```src/main/resources```, along with the framework's common proto definition.  You can download it from [here](http://git.covisintrnd.com/eng-core/protobuf-support/raw/master/src/main/resources/framework.proto).  The framework proto will need to be placed in the same location as the ones you will create.

5. In a terminal window ```cd``` to the proto file directory and issue command ```protoc --java_out=../java your_file.proto```.  This will generate all the Java source files and place them into the protobuf module's ```src/main/java``` source folder.

6. Delete the ```framework.proto``` that you copied into the working directory.  This was only necessary for compilation.  Finally, issue a ```mvn compile``` on the protobuf module to ensure that everything compiles without a problem.  You have now generated Google Protocol Buffer classes that will convert the generated POJOs to and from the protobuf binary protocol.  Congratulations!


_Note: Because the ```protobuf``` module contains generated source code, it will not follow Covisint's code standards and so to that effect, checkstyle and other code hygiene tools have been disabled for this module._

## Reader

All readers of protobuf representations must extend ```com.covisint.core.http.service.core.io.protobuf.AbstractProtobufResourceReader``` which itself implements ```EntityReader```.  If the resource being marshaled is realm-scoped (which most will be) then the reader will extend ```com.covisint.core.http.service.core.io.protobuf.AbstractRealmScopedProtobufResourceReader```.  You should name your reader class so that it is clear that it deals with the protobuf format.  For example, ```BookProtobufReader```.

Similar to the JSON readers, the protobuf reader will need to implement the ```#getResourceType```, ```#createResource``` and ```#isReadable``` methods.  The implementation details are exactly the same as those described in the section dealing with JSON readers.

The main method you will implement in your protobuf reader is:

```java
    // Constraint annotations and javadoc redacted.
    public MyResource populateResource(MediaType mediaType, MyProtoType proto,  MyResource resource) {
       ...

```

The ```#populateResource``` method implementation will read the protobuf data out of ```proto``` (which is whatever type you generated from your IDL source file) and populate that data into ```resource``` (which is the resource you are populating with the data).  As a convenience, you will return the populated resource.

Just like in the JSON readers, this method should do no validation or constraint checking and its primary purpose is simply to populate ```resource``` as best it can based on the data contained within ```proto```.

Somewhere at the top of your method implementation you should make a call to the parent class method ```#populateAbstractResourceFields```, passing in the core framework proto message.  This method will populate the core resource fields (id, realm, version, etc) automatically and save the developer from doing this again and again.  Here is an example of this method being used:

```java
  ...
  populateAbstractResourceFields(mediaType, proto.getCoreResource(), resource);
  ...
```

The other method you will need to implement for protobuf readers is ```#getMessageParser```.  This method must return an instance of ```com.google.protobuf.Parser``` and is available at the following location of all protobuf-generated classes, for example, ```MyProtoType.PARSER```.  So a typical method implementation will look like

```java
    // Constraint annotations and javadoc redacted.
    protected Parser<MyProtoType> getMessageParser() {
        return MyProtoType.PARSER;
    }
```
***

***
## __Helper Method: Reading International String__

In the JSON reader section we talked about the ```InternationalString``` common type and the helper methods available in the abstract reader to read this type.  There is a similar utility method available in the base protobuf reader class that will read an international string and return a ```java.lang.Iterable<InternationalString>```:

```java
    /**
     * Converts the given {@link ProtoInternationalString}s to the framework equivalent {@link InternationalString}.
     * 
     * @param internationalStrings The proto messages representing i18n strings.
     * @param parentResourceId The owning entity id for the i18n strings.
     * @return Returns an iterable of converted international strings.
     */
    @Nullable
    @NonnullElements
    public static final Iterable<InternationalString> readInternationalString(
            @Nullable List<ProtoInternationalString> internationalStrings, String parentResourceId) {
    ...
```

With this method you simply pass in a list of ```ProtoInternationalString```s and an optional parent resource id.  It will return an iterable of the framework's ```InternationalString``` type.  ```ProtoInternationalString``` is part of a common framework library ```protobuf-support``` and is the protobuf equivalent to the international string type.

***

## __Helper Method: Reading Resource References__

There is also a utility method that exists to help read resource references:

```java
    /**
     * Converts the given {@link ProtoResourceRef} to the framework equivalent {@link ResourceReference}.
     * 
     * @param resourceRef The proto message representing a resource reference.
     * @return Returns an instance of converted resource reference.
     */
    @Nullable
    public static final ResourceReference readResourceReference(@Nullable ProtoResourceRef resourceRef) {
    ...
```
***

This utility method will take a ```ProtoResourceRef``` (another class within ```protobuf-support```) and convert it into a ```ResourceReference```.

## Writer

All protobuf entity writers must extend ```com.covisint.core.http.service.core.io.protobuf.AbstractProtobufResourceWriter<T, P>``` which itself implements the interface ```com.covisint.core.http.service.core.io.EntityWriter```. The generic type ```T``` is the type of resource to write.  The type ```P``` is the protobuf type that will be used to actually serialize the data to binary format.  If the resource is realm scoped, then it will have to instead extend ```com.covisint.core.http.service.core.io.protobuf.AbstractRealmScopedProtobufResourceWriter```.  As with the JSON writer, the following methods are required to be implemented by the protobuf writer: ```#getResourceType``` and ```#isWritable```.  The implementation details are exactly the same as with the JSON flavor.

The main method in the protobuf writer is ```#writeResource```, which will actually write out the given resource to binary protobuf format.  Its signature is shown below:

```java
public void writeResource(MediaType mediaType, MyResource resource, OutputStream output)
            throws IOException {
   ...
```

The arguments to this method are very similar to its JSON writer counterpart, but the implementation is a bit different.  In the body, you will need to create the proto POJO and ultimately call ```MyProtoType.Builder.build().writeDelimitedTo(output)```.  The ```MyProtoType``` will be whatever protobuf class you generated from the IDL source file, and after you have fully populated the class you will need to get its builder, build it, and write it "delimited" to the output stream.  Here's an example of how this would be done:

```java
...
        // Create the proto object.
        MyProtoType.Builder proto = MyProtoType.newBuilder();

        proto.setCoreResource(buildCoreResource(resource));

        proto.setXyz(resource.getXyz());
        // Set other fields...

        // Now write the message to the output stream.
        proto.build().writeDelimitedTo(output);
```

In this case we are building an instance of ```MyProtoType``` via its ```Builder```.  An instance of the builder is created via ```#newBuilder```.  Next, a call is made to ```#buildCoreResource```; this method exists in the abstract writer class and will assist you by populating and resource a ```ProtoCoreResource``` object and returning it.  This object is set into your proto instance.  After calling all other required setters on your object, you will write it out to the output stream.

_Note: There are two methods protobuf gives you to write out your final proto object - ```#write``` and ```#writeDelimitedTo```.  We used the latter form because this resource may be part of a larger collection to be written out in response to a search request.  If we would use ```#write```, then protobuf will mark the end of the stream and we would only be able to write a single resource out in the response._

* Extension Point: Custom Formats

So far we have seen that the framework supports JSON and Google Protocol Buffer data structures.  There is nothing stopping a developer from using a custom data format, if absolutely necessary (although discouraged).  For example, if an XML format is needed, then simply create a ```BookXmlReader``` and ```BookXmlWriter``` and have them implement the appropriate reader and writer interfaces.  You will need to ensure that you take care of ALL aspects of the resource, such as reading/writing core resource properties, international strings, resource references, and so on.  Also take care to define a correct media type for such an extension.  In the case above, perhaps the media type would be ```application/vnd.com.covisint.platform.book.v1+xml```.

<table>
  <tr>
    <td><a href="json-readers-writers">&lt; 1.2. JSON Reader/Writer</a></td>
    <td align="right"><a href="core-configuration">1.4. Configuration &gt;</a></td>
  </tr>
</table>