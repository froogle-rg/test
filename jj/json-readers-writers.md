<table>
  <tr>
    <td><a href="resources-vs-entities">&lt; 1.1. Resources and Entities</a></td>
    <td align="right"><a href="proto-readers-writers">1.3. Protobuf Reader/Writer &gt;</a></td>
  </tr>
</table>
# 1.2. JSON Reader/Writer

Once the model classes are created it is time to think about how they will be marshaled.  During initial API design the entity media types will have been determined.  This step is where entity readers and writers will be written for each of those media types.

JSON is the _lingua franca_ of the web and is one of the HTTP body formats supported by the framework out of the box.  This section will describe the framework's support for JSON readers and writers, including what is provided by base classes and what can be extended and customized.

For the purposes of this section, lets assume we have the following media types to represent the JSON format of our authorization and book entities:

```
application/vnd.com.covisint.platform.book.v1+json

application/vnd.com.covisint.platform.authorization.request.v1+json
application/vnd.com.covisint.platform.authorization.response.v1+json
```

Here we have a media type representing v1 book representations in the JSON format, and media types for authorization request/response entities, also in JSON format.

For each media type a reader and a writer is required.  The reason for both is as follows:

1. Client makes a request to the server.  Client will use the entity writer to write the request body.  Server will use the entity reader to read that request body.

2. Server sends a response to the client.  Service will use the entity writer to write the response body.  Client will use the entity reader to read the response body.

How media types are used in HTTP headers as part of content negotiation are beyond the scope of this document.  Consult the appendix for HTTP RFC references where you can learn all about them.

Now that we have covered the general concepts, lets dive into the code.

## Package Naming Convention

The convention for naming JSON marshaler packages is as follows:

```
com.covisint.platform.{serviceId}.core.{entityOrResourceType}.io.json
```

For example:

```
com.covisint.platform.book.core.book.io.json
```

This package will contain all classes that read and write the book resources to and from JSON.  If we were creating the package to contain our authorization request/response entity marshalers, the package should be named

```
com.covisint.platform.authorization.core.authorization.io.json
```

## Entity Reader

All entity readers must implement ```com.covisint.core.http.service.core.io.EntityReader<T>``` where ```T``` is the type of entity being read.  This interface declares the following methods:

```java
    boolean isReadable(com.google.common.net.MediaType mediaType);

    T read(com.google.common.net.MediaType mediaType, java.io.InputStream input);

    java.lang.Class<T> getResourceType();
```

* __isReadable()__ returns a flag indicating whether the reader can handle the given media type.  It is very possible that a single reader class can support more than one media type.  Keep in mind that the passed media type may also contain custom attributes, so this too can be used as part of the evaluation.

* __read()__ is guaranteed to be called if and only if ```#isReadable``` returns true.  The supported media type is provided as an argument, in addition to an input stream into the entity body.  This method will return a type ```T``` which is the populated entity that was read from the input stream.

* __getResourceType()__ is a convenience interface method of sorts and returns the concrete implementation class.

If you are reading entity POJOs and __NOT__ proper resources, then you will need to implement this interface directly and read the entire entity yourself.  For example if we were creating an entity reader for an authorization request entity it may look something like this:

```java
 
    import javax.json.Json;
    import javax.json.JsonStructure;
    import javax.json.JsonReader;
    import javax.json.JsonException;
    import javax.json.stream.JsonParsingException;
    import com.covisint.core.http.service.core.io.EntityParsingException;

    ...

    // javadoc and constraint annotations omitted for brevity.
    public AuthorizationRequest read(MediaType mediaType, InputStream input) {

        try (final JsonReader jsonReader = Json.createReader(input)) {
            final JsonStructure json = jsonReader.read();

            return ...// read JSON structure into AuthorizationRequest and return

        } catch (JsonParsingException e) {
            throw new EntityParsingException(e.getLocation().getLineNumber(), e.getLocation().getColumnNumber(), e);
        } catch (JsonException e) {
            throw new EntityReaderException("Unable to read JSON");
        }

    }
```

This is the template for all entity reader implementations of the ```#read``` method.  You should always handle the case where the input is not valid JSON and that is where an ```EntityParsingException``` should be thrown.  A catch-all would be if the courser-grained ```JsonException``` is thrown in which case a general ```EntityReaderException``` is triggered.

## Resource Reader

Working with resources is a bit easier than rolling your own implementation of a complete entity reader.  For that, the framework provides an abstract class which implements the interface: ```com.covisint.core.http.service.core.io.jsonp.AbstractResourceReader<T extends AbstractResource>```.  In this class, the generic type ```T``` must extend the framework's ```AbstractResource```, which your resource will have to extend.  The full class signature for the abstract resource reader is

```java
public abstract class AbstractResourceReader<T extends AbstractResource> implements EntityReader { ...
```

Resources that are __NOT__ realm-scoped can directly extend this abstract class.  However, if your resources __ARE__ realm-scoped (which most should be) then there is a more appropriate class you must extend - ```com.covisint.core.http.service.core.io.jsonp.AbstractRealmScopedResourceReader<T extends AbstractRealmScopedResource>```.  This class is used to read realm-scoped resources, and in fact the generic type ```T``` does extend ```AbstractRealmScopedResource``` which means your realm-scoped resource must also do the same.  The full class signature for the abstract realm-scoped resource reader is

```java
public abstract class AbstractRealmScopedResourceReader<T extends AbstractRealmScopedResource> extends
        AbstractResourceReader<T> { ...
```

These parent classes take care of some of the basics for you such as opening and closing the stream, throwing errors where needed, reading the core resource fields (id, version, etc).  They also provide some common utilities that you may need in the future such as reading a ```com.covisint.core.http.service.core.ResourceReference``` or collections of ```com.covisint.core.http.service.core.InternationalString```s.  These two types are covered a little further down.

Lets take a look at a sample resource reader for the book v1 JSON media type mentioned earlier:

```java
package com.covisint.platform.book.core.book.io.json;

import com.covisint.core.http.service.core.io.jsonp.AbstractRealmScopedResourceReader;
import com.covisint.service.book.core.book.Book;
import com.google.common.net.MediaType;
import javax.json.JsonObject;

...

// javadoc and constraint annotations omitted for brevity
public class BookJsonReader extends AbstractRealmScopedResourceReader<Book> {

    protected Book createResource(MediaType mediaType) {
        return new Book();
    }

    public boolean isReadable(MediaType mediaType) {
        return // true if media type can be read, otherwise false
    }

    protected Book readResource(MediaType mediaType, JsonObject json) {

        final Book book = super.readResource(mediaType, json);

        // continue reading book properties...

        return book;
    }

}
```

These three methods require overriding from the parent class.  Lets break them down to understand what they do:

__createResource()__: This method is quite simple.  It returns a new, unpopulated instance of the resource based on the given media type.  In this case, the reader can only read book resources and so there is no determination made; it simple returns a new ```Book```.  In other cases though, there may be a few subclasses of book that may be read by this reader.  In that case possibly the media type would make that determination, and this is where the logic would go.

__isReadable()__: This method determines if the reader class can read the given media type.  The framework guarantees that it will call this method before proceeding to call ```#createResource``` or ```#readResource```.

__readResource()__: This is the main method in the reader class and is responsible for populating a ```Book``` instance based on the given JSON object.  The first step of this method should always make a call to the superclass method to get a starting ```Book``` instance pre-populated with the core resource fields.  At that point, only the custom Book-specific properties need to be added, simplifying the work and implementation of the reader class.  Another point to note here is that the media type is passed into this method.  In the case of the book reader example above, the media type isn't really needed and is ignore.  However there may be cases where a reader supports multiple media types.  Here is where the checks would need to be made so that the JSON content is properly parsed and understood.

_IMPORTANT: As a rule of thumb, your readers should be "dumb".  No business logic should exist within them, and their sole purpose is to try their best to take the given JSON and populate the Java resource object.  Do not perform validation within readers!  Do not throw exceptions (if at all possible) within readers!  That is the job of the validation layer._

***

## __Helper Method: Reading Resource References__

In the framework there is a type ```com.covisint.core.http.service.core.ResourceReference``` which is a POJO that encapsulates an id and type and an optional realm.  This class represents a reference to an external resource.  You can use it as a foreign key of sorts when a resource has a reference to some other resource, whether that resource is owned by the current service or some other service.  The ```id``` property is the referenced resource's id.  The ```type``` property indicates the resource type and ```realm``` indicates the realm in which the resource exists.  A realm specifier is helpful in cases where the referenced resource actually exists in another realm (although this is not going to be a common occurrence).  There is a standard JSON schema which describes the resource reference object, and they look like this:

```json
{
  "id": "123",
  "type": "book",
  "realm": "ABC"
}
```

To help eliminate code duplication and manual effort on behalf of the developer, there is a utility method that exists to help read resource references:

```java
ResourceReference com.covisint.core.http.service.core.io.jsonp.AbstractResourceReader.readResourceReference(JsonObject json)
```
From within your resource reader you can call ```#readResourceReference``` and pass it the JSON object that contains the resource reference to read.  It will return you a populated ```ResourceReference```, assuming the JSON was populated as expected.
***

***
## __Helper Method: Reading International String__

In the framework there is a type ```com.covisint.core.http.service.core.InternationalString``` that represents a common way to write i18n strings in multiple languages.  This approach is good practice so as to promote a homogenous API design.  International strings are documented in the common JSON API schema and look like this in JSON format:

```json
{
...

  "name": [
    { "lang": "en", "text": "My name is Sam" },
    { "lang": "el", "text": "Το όνομά μου είναι Σαμ" },
    { "lang": "fr", "text": "Je m'appelle Sam" },
    { "lang": "zh", "text": "我的名字是山姆" }
  ]

...
}
```

This is one property - ```name``` - that is available in four translations; english, greek, french and chinese.

There is a utility method available in the base reader class that will read an international string and return a ```java.lang.Iterable<InternationalString>```:

```java
    /**
     * Reads an internationalized string from the given JSON content.
     * 
     * @param json The JSON containing the data to read.
     * @param owningEntityId The id of the entity that owns this string.
     * @param langProp The name of the JSON property carrying the language.
     * @param textProp The name of the JSON property carrying the text value.
     * @return Returns an iterable of i18n string objects.
     */
    public static final Iterable<InternationalString> readInternationalString(JsonArray json,
            String owningEntityId, String langProp, String textProp) { ...
```

```owningEntityId``` is the id of the entity or resource that contains the international string.  If we wanted to add an international string property to our book resource, then the owning entity id would be the id of the book resource we are reading.  This field is optional and you can pass null if you don't have one available to you while reading.
***

## Entity Writer

All entity readers must implement ```com.covisint.core.http.service.core.io.EntityWriter<T>``` where ```T``` is the type of entity being written.  This interface declares the following methods:

```java
    boolean isWritable(Class<?> clazz, com.google.common.net.MediaType mediaType);

    com.google.common.net.MediaType write(com.google.common.net.MediaType mediaType, 
             T source, java.io.OutputStream output);

    java.lang.Class<T> getResourceType();
```

* __isWritable()__ returns a flag indicating whether the writer can write the given media type and/or class.  A single writer can support more than one media type and may also be able to process more than one class (for example, it may be able to write out one of several subclasses).  Keep in mind that the passed media type may also contain custom attributes, so this too can be used as part of the evaluation.

* __write()__ is guaranteed to be called if and only if ```#isWritable``` returns true.  The supported media type is provided as an argument, in addition to an output stream used to write out the body.  The source object is passed in, which is the actual entity object being written.  This method will return a media type which is _usually_ the passed in media type, but may possibly be altered with some additional parameters depending on how the write occurred.

* __getResourceType()__ is a convenience interface method of sorts and returns the concrete implementation class of the writer.

If you are writing entity POJOs and __NOT__ proper resources, then you will need to implement this interface directly and write the entity yourself.  An entity writer for an authorization response entity may look like:

```java

    import com.google.common.net.MediaType;
    import javax.json.Json;
    import javax.json.JsonException;
    import javax.json.JsonWriter;
    import javax.json.JsonWriterFactory;
 
    public MediaType write(MediaType mediaType, AuthorizationResponse entity, java.io.OutputStream output) {
        
        final JsonWriterFactory jsonFactory = Json.createWriterFactory(new java.util.HashMap<String, Object>());
        
        try (final JsonWriter writer = jsonFactory.createWriter(output, Charsets.UTF_8)) {
            
            final JsonObject json = ...; // Convert entity to JSON.
            writer.write(json);

            return mediaType;
        } catch (JsonException e) {
            throw new EntityWriterException("Unable to write JSON");
        }

    }
```

This is the template for all entity writer implementations of the ```#write``` method.  You will create a ```javax.json.JsonWriter``` from within a try-with-resources block and then write out the JSON-converted entity (```AuthorizationResponse``` in this case).  Catch any ```JsonException``` that may occur and re-throw it as a ```EntityWriterException``` so the framework can relay that error back to the client.

## Resource Writer

In the case where you are writing actual resources to a JSON output, there are framework-provided base classes at your disposal...

```com.covisint.core.http.service.core.io.jsonp.AbstractResourceWriter<T extends AbstractResource>``` is a base writer class that can process any resource that extends from ```com.covisint.core.http.service.core.AbstractResource```.  If you have resources that are __NOT__ realm scoped, then they will extend this abstract resource class and likewise your writers will extend ```AbstractResourceWriter```.

If your resources __ARE__ realm-scoped then your writer will instead extend ```com.covisint.core.http.service.core.io.jsonp.AbstractRealmScopedResourceWriter<T extends AbstractRealmScopedResource>``` which can handle any ```AbstractRealmScopedResource```.  Lets see a code sample of a resource writer that operates on book resources.  It will extend the realm-scoped resource writer since books are realm-scoped:

```java
// javadoc and constraint annotations omitted for brevity.
public class BookJsonWriter extends AbstractRealmScopedResourceWriter<Book> {

    public boolean isWritable(Class clazz, MediaType mediaType) {
        return ...; // Check to see if this writer can handle the class and media type given
    }

    protected JsonObjectBuilder writeResource(MediaType mediaType, Book book) {

        final JsonObjectBuilder builder = super.writeResource(mediaType, book);

        // Write out all book properties to the builder object.

        return builder;
    }

    public Class<Book> getResourceType() {
        return Book.class;
    }
}
```

These three methods require overriding from the parent class. Lets break them down to understand what they do:

__isWritable()__: Determines if the writer can write resources of the given media type and class.  If so, this method returns true; false otherwise.  The media type may contain parameters that request the resource to be written a certain way.  Pay attention to these as they are part of the client/server content negotiation.  This method is guaranteed to be called prior to ```#writeResource```.

__writeResource()__: This is the main method in the resource writer and is responsible for converting the given resource object to its JSON format, ultimately building and returning a JSON builder object.  A call to the super class method will start a ```JsonObjectBuilder``` and pre-populate it with the core resource properties.  At that point you are free to append all the rest of the resource's data to the builder and return it when done.  The framework will then finish processing and write the entire response body out to the stream.

__getResourceType()__: This method returns the concrete implementation class of the resource writer.  Pretty straight forward.

_IMPORTANT: As a rule of thumb, your readers should be "dumb". No business logic should exist within them, and their sole purpose is to try their best to take the given entity POJO and populate the JSON builder object. Do not perform validation within writers! Do not throw exceptions (if at all possible) within writers!_

***

## __Helper Method: Writing Resource References__

Just like we saw in the JSON reader section, there are utilities to make writing resource references easier on the developer.  It saves time, energy and bugs.  Since the writer's job is to take a POJO and convert it to JSON, this utility method will take a reference to a ```com.covisint.core.http.service.core.ResourceReference``` and will add it to a JSON object builder that is taken to be the "containing" JSON of that reference.

The method signature is shown here:

```java
void com.covisint.core.http.service.core.io.jsonp.AbstractResourceWriter.writeResourceReference(String refName, ResourceReference reference, JsonObjectBuilder parentJson);
```

It takes a ```refName``` which is the JSON property "key" under which the reference will be written.  And as expected, it expects you to provide the resource reference and the parent JSON.  It will marshal the resource reference to the standard JSON format and append that JSON snippet into the parent JSON.  Again, ```id``` and ```type``` are mandatory fields, whereas ```realm``` is optional.  If realm is not supplied the method will not write anything out for that property.

***
***
## __Helper Method: Writing International Strings__

If a resource contains an i18n string - a collection of ```InternationalString``` objects - there is a utility method available from within any resource writer that will allow easy marshaling of those strings to their JSON format...as documented in the common JSON schema.

```java
void com.covisint.core.http.service.core.io.jsonp.AbstractResourceWriter.writeInternationalString(String propertyName, Iterable<InternationalString> strings, String keyName, String valueName, @Nonnull JsonObjectBuilder builder)
```

```propertyName``` is the name of the JSON property under which the i18n string array will be written.  For example, if a book resource had a ```name``` property which was translated in multiple languages, then the value would be ```name```.

An iterable of InternationalString objects is passed, as well as ```keyName``` and ```valueName```.  These are always passed as ```lang``` and ```text``` according to the JSON schema definition.

Finally, the JSON object builder is required so that the method can append the international strings to it in JSON format.

***

<table>
  <tr>
    <td><a href="resources-vs-entities">&lt; 1.1. Resources and Entities</a></td>
    <td align="right"><a href="proto-readers-writers">1.3. Protobuf Reader/Writer &gt;</a></td>
  </tr>
</table>