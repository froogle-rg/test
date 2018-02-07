<table>
  <tr>
    <td><a href="client-builder">&lt; 4.2. Builder</a></td>
    <td align="right"><a href="client-factory">4.4. Factory &gt;</a></td>
  </tr>
</table>

# 4.3. Client Implementation

We now come to the end (well, almost) of the Internal Client explanation. Till now, you have a client interface and a client builder i.e., `BookClient` and `BookClientBuilder`. Let's now add the final ingredient aka `BookClientImpl` which completes the Internal Client implementation.

The client implementation is nothing but the implementation of the previously described client interface. We've seen that the client interface defines several methods for CRUD and Search. Now in your implementation, it is obvious that you have to implement all of these methods. Seems like a lot, ain't it? But like always, the framework saves you from all that effort. It provides an abstract class called `BaseResourceClient` which implements all of these methods. If you remember, we talked about Executor Service, HTTP Client, entity readers and writers and more in the Client Builder. Now all that will come into action in the Client Implementation. The `BaseResourceClient` constructs the CRUD and Search URIs, uses the entity readers and writers, creates and executes the HTTP Requests asynchronously and returns a future as the response. We will not go into the details of how each of these methods are implemented, since you will not need it much. But if you would want to understand further, we suggest you go through the code [here](http://git.covisintrnd.com/eng-core/http-service-framework/blob/master/client/src/main/java/com/covisint/core/http/service/client/BaseResourceClient.java).

## Your Client Implementation

Like we explained in the previous sections, we need to create an interface and builder for each resource. Similarly, you will have to create a client implementation for your resource. A simple client implementation with basic CRUD and Search endpoints will look like this:

```java
/** Resource client implementation that operates on {@link Book}s. */
public class BookClientImpl extends BaseResourceClient<Book> implements BookClient {

}
```

There are no methods since all of it is provided by the parent class `BaseResourceClient`. 

***
__Extension Point: Custom endpoints__

But like we explained previously, if you have custom endpoints like /tasks/* or hierarchical URIs or custom media type parameters, then you will have to implement the corresponding methods in your class. 

Continuing the library example, you will have to create the attributes to hold the custom endpoints. You should also create their setter methods. The following code snippet explain this to you:

```java
/** Resource client implementation that operates on {@link Book}s. */
public class BookClientImpl extends BaseResourceClient<Book> implements BookClient {
    /** The borrow-book URI template. */
    private UriTemplate boorowBookUriTemplate;

    /** The return-book URI template. */
    private UriTemplate returnBookUriTemplate;

    /**
     * Sets the URI template for the borrow book endpoint.
     * 
     * @param template The template to set.
     * @return Returns this instance.
     */
    @Nonnull
    protected BookClientImpl setBorrowBookUriTemplate(@Nonnull UriTemplate template) {
        boorowBookUriTemplate = template;
        return this;
    }

    /**
     * Sets the URI template for the return book endpoint.
     * 
     * @param template The template to set.
     * @return Returns this instance.
     */
    @Nonnull
    protected BookClientImpl setReturnBookUriTemplate(@Nonnull UriTemplate template) {
        returnBookUriTemplate = template;
        return this;
    }
}
```  

If you remember from the previous chapter, the `#setBorrowBookUriTemplate` and `#setReturnBookUriTemplate` methods were used in the builder's `#build` method.

Now, you will now have to implement all the custom endpoint methods defined in the interface i.e., `#borrowBook` and `#returnBook`.

```java
    ...

    /** {@inheritDoc} */
    @Nonnull
    public CheckedFuture<Void, ServiceException> borrowBook(@Nonnull @NotEmpty String bookId,
            @Nonnull HttpContext httpContext) {
        try {
            final Map<String, Object> args = ImmutableMap.<String, Object> of("bookId", bookId);
            final String uri = boorowBookUriTemplate.expand(args);
            return execute(new HttpPost(uri), MediaType.ANY_TYPE, httpContext);
        } catch (VariableExpansionException | MalformedUriTemplateException e) {
            throw new ClientException("Error occurred while expanding URL template.", e);
        }
    }

    /** {@inheritDoc} */
    @Nonnull
    public CheckedFuture<Void, ServiceException> returnBook(@Nonnull @NotEmpty String bookId,
            @Nonnull HttpContext httpContext) {
        try {
            final Map<String, Object> args = ImmutableMap.<String, Object> of("bookId", bookId);
            final String uri = returnBookUriTemplate.expand(args);
            return execute(new HttpPost(uri), MediaType.ANY_TYPE, httpContext);
        } catch (VariableExpansionException | MalformedUriTemplateException e) {
            throw new ClientException("Error occurred while expanding URL template.", e);
        }
    }

    ...
```

As you can see, once you create the URI using the URI templates, you can execute the HTTP requests. You can make use of the `#execute` methods in the `BaseResourceClient` class.

An example of a customized client implementation can be found [here](http://git.covisintrnd.com/eng-iot-core/device-service/blob/master/client/src/main/java/com/covisint/platform/device/client/device/DeviceClientImpl.java)

***

***
__Extension Point: Client-side caching__

There are times when the consumers may need to configure client-side caching. The client supports the configuration of this. By default, caching is not enabled. But in case you want to enable this, then you will have to create a `CacheSpec` and configure the client accordingly. You can see the cache configuration  in ```com.covisint.core.http.service.client.BaseResourceClientFactory```'s `#configureCache` method.

Here is the sample code to create a CacheSpec:

```java
        CacheSpec.Builder cacheConfigBuilder = CacheSpec.builder();
        cacheConfigBuilder.expiration(3000, ExpirationMode.AFTER_WRITE);
        cacheConfigBuilder.maxElements(1000);
        CacheSpec cacheConfig = cacheConfigBuilder.build();
        bookClient.configureCache(cacheConfig);
```

*NOTE: Client-side caching is enabled for get-by-id and search endpoints. In case update / delete endpoints are used, then cache eviction doesn't happen automatically. You should carefully handle this with the right configuration.*

***

## How to use the Internal Client

Now that we've seen all the different components of the Internal Client, let us see some sample code which explains how to create a client instance and use it.

```java
        // Create a HTTP client making use of the HttpClientBuilder

        final HttpClientBuilder builder = new HttpClientBuilder().setMaxConnections(1024)
                .setConnectionClosedAfterResponse(true);

        // Now add the necessary intercepters which will read values off the request header
        builder.addSpanInterceptor("X-Span", "X-Trace");
        builder.addXRealmInterceptor();
        builder.addXRequestorInterceptor();
        builder.addXRequestorApplicationInterceptor();

        final HttpClient httpClient = builder.build();

        // Create the executor service used to execute the requests asynchronously
        final ListeningExecutorService executor = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(10));

        // Replace <uri> with the actual uri of the microservice
        final String url = "http://<uri>"; 

        // Now create your book client builder
        final BookClientBuilder bookClientBuilder = new BookClientBuilder().setExecutorService(executor)
                .setHttpClient(httpClient).setServiceBaseUrl(url);

        bookClientBuilder.addEntityReader(new HttpServiceErrorReader<HttpServiceError>())
                .addEntityReader(new BookJsonReader()).addEntityReader(new BookProtobufReader());
        bookClientBuilder.addEntityWriter(new HttpServiceErrorWriter<HttpServiceError>())
                .addEntityWriter(new BookJsonWriter()).addEntityWriter(new BookProtobufWriter());

        // Create an instance of BookClient
        final BookClient client = bookClientBuilder.build();

        // Now you can call APIs by calling the client methods.

        // Create the HTTP Context
        final HttpContext context = new BasicHttpContext();
        context.setAttribute(XRealmInterceptor.CONTEXT_ATTRIBUTE_NAME, "ABC");
        context.setAttribute(XRequestorInterceptor.CONTEXT_ATTRIBUTE_NAME, "Sam");
        context.setAttribute(XRequestorApplicationInterceptor.CONTEXT_ATTRIBUTE_NAME, "Sam's App");

        // Let us call the get-by-id method
        CheckedFuture<Book, ServiceException> result = client.get(bookId, httpContext);
        Book book = result.get();
```

And we are done. You can call any client method on similar lines. Just looking back at the code snippet again, you feel like it's a lot of configuration you have to do to create a simple client right? Hold your breath! Things will get easier when you see the [Factory](client-factory) in the next chapter. The factory is not just meant for Public clients. It can also be used for Internal clients. Read on!

<table>
  <tr>
    <td><a href="client-builder">&lt; 4.2. Builder</a></td>
    <td align="right"><a href="client-factory">4.4. Factory &gt;</a></td>
  </tr>
</table>