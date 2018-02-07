<table>
  <tr>
    <td><a href="client-impl">&lt; 4.3. Implementation</a></td>
    <td align="right"><a href="client-sdk">4.5. SDK &gt;</a></td>
  </tr>
</table>

# 4.4. Client Factory

As promised in the previous chapter, we are now going to see how a Client Factory can make things easier for configuring a client. The framework has an abstract class called `BaseResourceClientFactory`. This has all the logic to create a client instance using the client builder and the base URL of the service. We will not go into specifics of the `BaseResourceClientFactory` here. If you want to explore all the code in there, you can see the code [here](http://git.covisintrnd.com/eng-core/http-service-framework/blob/master/client/src/main/java/com/covisint/core/http/service/client/BaseResourceClientFactory.java). 

What we will tell you is how to create your factory and how to use that to create your client.

## Your Client Factory

Creating you client factory is extremely simple. Extend your class from `BaseResourceClientFactory` and implement the abstract methods `#newBuilder` and `#buildResourceClient`.  See the code snippet below and explanation will follow:

```java
public class BookClientFactory extends BaseResourceClientFactory<BookClientBuilder, BookClientImpl> {

    /**
     * Constructor.
     *
     * @param serviceUrl the base url.
     */
    public BookClientFactory(@Nonnull @NotEmpty String serviceUrl) {
        super(serviceUrl);
    }

    /** {@inheritDoc} */
    @Nonnull
    protected final BookClientBuilder newBuilder() {
        return new BookClientBuilder();
    }

    /** {@inheritDoc} */
    @Nonnull
    protected final BookClientImpl buildResourceClient(@Nonnull BookClientBuilder builder) {
        builder.addEntityReader(new HttpServiceErrorReader<HttpServiceError>()).addEntityReader(new BookJsonReader())
                .addEntityReader(new BookProtobufReader());
        builder.addEntityWriter(new HttpServiceErrorWriter<HttpServiceError>()).addEntityWriter(new BookJsonWriter())
                .addEntityWriter(new BookProtobufWriter());
        return builder.build();
    }
}
```

It is as simple as this. You just need the constructor and implementation of the abstract methods.

1. `#BookClientFactory` - This is the constructor. It takes the base URL of the service and passes it on to the parent.

2. `#newBuilder` - This methods just creates an instance of the Client Builder we saw previously and returns it. This Client Builder will be used by the Factory to create an instance of the client.

3. `#buildResourceClient` - This method takes an instance of the Client Builder. You will just have to add your entity readers and writers into the builder. After this is done, just build it and your client instance is ready for use.


## How to use the Client Factory

Let us now see how to use this Client Factory:

```java
        // Replace <uri> with the actual uri of the microservice
        final String url = "http://<uri>"; 

        // Create an instance of BookClient using the client factory
        final BookClient client = new BookClientFactory(url).create();

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

You can see how the amount of code to configure the client has drastically reduced.
 
<table>
  <tr>
    <td><a href="client-impl">&lt; 4.3. Implementation</a></td>
    <td align="right"><a href="client-sdk">4.5. SDK &gt;</a></td>
  </tr>
</table>