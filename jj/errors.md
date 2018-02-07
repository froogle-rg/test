<table>
  <tr>
    <td><a href="span-trace-id">&lt; 12. Span/Trace IDs</a></td>
    <td align="right"><a href="guidelines">VI Guidelines and Standards &gt;</a></td>
  </tr>
</table>
# 13. Errors

All 400- and 500-level HTTP responses are represented as small JSON entities containing three properties:

1.  status: The HTTP response code
2.  apiStatusCode: An API-specific status code string.  This is fully namespaced and indicates the class/category of error that occurred.  This value should be listed and explained in API documentation so that consumers can code against it if needed.
3.  apiMessage: A user-friendly message that may be displayed to the caller.  Note that this is <b>NOT</b> internationalized and so should not be relayed to any final UI.

The framework does not provide any other representation format option for errors; only JSON.

Here is an example 404 error response sent by the service:

```json
{
  "status": 404,
  "apiStatusCode": "framework:resource:missing",
  "apiMessage": "A resource with the following ID was not found: 6395ac7a"
}
```

## I/O

Error responses are payloads that need to be written and read by the framework just like any other entity.  Therefore you will need to tell the framework which entity marshalers will be used to do this.  

For servers, specify them in the spring-view.xml:

```xml
  <util:list id="entityReaders">
      <bean class="com.covisint.core.http.service.core.io.jsonp.HttpServiceErrorReader" />
      ...
  </util:list>

  <util:list id="entityWriters">
      <bean class="com.covisint.core.http.service.core.io.jsonp.HttpServiceErrorWriter" />
      ...
  </util:list>
```

For clients, add them using the ```com.covisint.core.http.service.client.BaseResourceClientBuilder.addEntityReader(EntityReader)``` method, like this:

```java
 final MyBuilder builder = ... // set up builder
 builder.addEntityReader(new HttpServiceErrorReader<HttpServiceError>());
 builder.addEntityWriter(new HttpServiceErrorWriter<HttpServiceError>());
```

The framework will use these marshalers to process all errors that occur, whether the server is writing them or the client is reading them.

_Note that you should declare the error reader & writer first when building the entity marshaler lists, so that the framework will resolve them first when it encounters a service exception.  Otherwise it will attempt content negotiation which isn't desirable in this scenario._

## Server-Side
All errors related to the client's request should be thrown as 400-level errors.  All others can be classified as unhandled server-side errors and should be thrown as 500 errors.  Here are some of the more common exception types you will use to respond to these conditions:

| Class                                                              | status | apiStatusCode           | Description                             |
| :----------------------------------------------------------------- | :----: | :---------------------- | --------------------------------------- |
| ```c.c.c.h.s.c.i.EntityReaderException``` | 400    | ```framework:io:read``` | An error occurred while reading the request entity body. |
| ```c.c.c.h.s.c.i.EntityParsingException``` | 400    | ```framework:io:read:parsing``` | Could not parse the request entity. |
| ```c.c.c.h.s.c.i.EntityWriterException``` | 400    | ```framework:io:write``` | An error occurred while writing the response entity body. |
| ```c.c.c.h.s.s.c.MissingBodyException``` | 400    | ```framework:request:body:missing``` | The request did not contain the resource. |
| ```c.c.c.h.s.s.c.MissingHeaderRequestException``` | 400    | ```framework:request:header:missing``` | The request did not contain a required header. |
| ```c.c.c.h.s.s.c.MissingParameterRequestException``` | 400    | ```framework:request:param:missing``` | The request did not contain a required parameter. |
| ```c.c.c.h.s.s.c.UnsupportedHttpMethodException``` | 405    | ```framework:request:method:unsupported``` | The HTTP verb is not supported on the endpoint. |
| ```c.c.c.h.s.s.c.InvalidRequestException``` | 400    | ```framework:request``` | An unspecified request error occurred. |
| ```c.c.c.h.s.s.InvalidResourceException``` | 400    | ```framework:request:resource``` | There was an unspecified error with the resource. |
| ```c.c.c.h.s.s.EntityNotFoundException``` | 404    | ```framework:request:resource:missing``` | The requested resource(s) was not found. |
| ```c.c.c.h.s.s.IllegalDataResourceException``` | 400    | ```framework:request:resource:data:illegal``` | The resource contained illegal data. |
| ```c.c.c.h.s.s.MissingDataResourceException``` | 400    | ```framework:request:resource:data:missing``` | The resource did not contain all required data. |
| ```c.c.c.h.s.s.ResourceIdConflictException``` | 409    | ```framework:resource:conflict:id``` | The operation being requested would put the resource in a conflicted state. |
| ```c.c.c.h.s.s.NotAuthorizedException``` | 403    | ```framework:security``` | Unauthorized access. |
| ```c.c.c.h.s.s.DataAccessException``` | 500    | ```framework:dao``` | Data access error occured. |
| ```c.c.c.h.s.c.ServiceException``` | 500    | ```framework``` | Unhandled server error. |


All framework exceptions extend ```com.covisint.core.http.service.core.ServiceException``` which itself extends ```java.lang.RuntimeException```.  This means that you need not (and should not) declare any framework exception type in your method signatures.  

***
__Extension Point__

While not usually required, you may extend any of the framework exception types and use them in your service; just make sure to register an ```@org.springframework.web.bind.annotation.ExceptionHandler``` that can process the custom type.  See https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ExceptionHandler.html for more details.

***

## Client-Side

So far we have seen how the server processes requests and issues appropriate error HTTP responses.  In the client there is also a standard way that these errors get communicated and parsed.

The ```com.covisint.core.http.service.client.ResourceClient``` interface shows that all methods return a ```com.google.common.util.concurrent.CheckedFuture<T, ServiceException>```, where the second type parameter is of type ServiceException.  What this does is allow the consumer to wrap the invocation in a try/catch block, catch that exception type, and therefore do something like the following:

```java

    try {
        return myClient.get(resourceId, context).checkedGet();
    } catch (final ServiceException e) {
        if (e.getServiceStatusCode().equals("framework:resource:missing")) {
          // Process missing resource scenario.
        } else {
          // Handle all other scenarios.
        }
    }

```

You can check for any apiStatusCode in the catch block as desired.

<table>
  <tr>
    <td><a href="span-trace-id">&lt; 12. Span/Trace IDs</a></td>
    <td align="right"><a href="guidelines">VI Guidelines and Standards &gt;</a></td>
  </tr>
</table>