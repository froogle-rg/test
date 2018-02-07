<table>
  <tr>
    <td><a href="client-interface">&lt; 4.1. Interface</a></td>
    <td align="right"><a href="client-impl">4.3. Implementation &gt;</a></td>
  </tr>
</table>
# 4.2. Client Builder

Now that we have seen what a Client Interface is, we need to create a client instance. But before that, let's see the Client Builder offered by the framework which helps in creating the client instances with all the necessary configuration needed.

The framework has an abstract class called `BaseResourceClientBuilder` which has the ability to build a your client. Just like any other builder, it has a set of attributes and the corresponding get* and set* methods, and a method which finally builds the client.

## Attributes

Each of the following attributes will have their corresponding get* and set* methods:

1. `httpClient`: The [Apache HTTP Client](https://hc.apache.org/httpcomponents-client-ga/index.html) which is used to make the API requests.
   
2. `executor`: An [Executor Service](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html) to asynchronously execute the requests.

3. `serviceBaseUrl`: This is the base URL of your microservice. Based on this base URL, all the CRUD and Search endpoints are constructed.

4. `entityReaders`: This is the list of all entity readers used by the client to read the response of the requests. You must include the framework's `HttpServiceErrorReader` to read any error responses and all the JSON and Protobuf readers. 

5. `entityWriters`: This is the list of all entity writers used by the client to produce the request body. You must include the framework's `HttpServiceErrorWriter` to read any error requests and all the JSON and Protobuf writers

6. `cacheSpec`: This is optional. In case you want client side caching for your clients, then you will need to set this.

7. `proxySpec`: This is optional. In case you need to sepcify a HTTP connection proxy, then you can use this.

## Build utility methods

Let us now look at the important methods which actually build the client:

1. `abstract C build()`: This is an abstract method which has to be implemented by your client builder. It returns an instance of the client.

2. `abstract String getResourceCollectionPath()`: This is an abstract method to be implemented by your client builder. It gets the path to the base collection of resources. This is generally just a single path component that may, or may not, contain preceding or proceeding "/"'s. For example: /passwords, /users, /path/to/groups

3. `abstract MediaType getResourceRepresentation()`: This again is an abstract method to be implemented by your client builder. Gets the media type that the client will use for sending requests. Basically this will be the content type for the CRUD endpoints

4. `setCrudEndpoints(final C client)`: This method is used by the `#populateBaseBuilder` to construct and set the CRUD and Search endpoints into the client

5. `C populateBaseBuilder(C client)`: This method actually builds the client. It takes an instance of the client and populates the following into the client: 
    * list of entity readers
    * list of entity writers
    * the HTTP Client
    * the Executor service
    * the Content type of the requests
    * the Cache spec, if provided
    * the constructed CRUD and Search endpoints

## Your Client Builder

Each of your resource should have its own client builder. You can extend from the `BaseResourceClientBuilder` to get all the build functionalities out-of-the-box from the framework. You only have to implement the above mentioned abstract methods. Here is an example code of a simple client builder:

```java
/** Builder of {@link BookClient} clients. */
public class BookClientBuilder extends
        BaseResourceClientBuilder<BookClientBuilder, BookClientImpl> {

    /** Base path to the resource collection endpoint. */
    private static final String RESOURCE_COLLECTION_PATH = "/books";

    /** {@inheritDoc} */
    @Nonnull
    @NotEmpty
    protected String getResourceCollectionPath() {
        return RESOURCE_COLLECTION_PATH;
    }

    /** {@inheritDoc} */
    @Nonnull
    protected MediaType getResourceRepresentation() {
        return MediaType.parse(MediaTypes.BOOK_V1_PROTO_MT.value());
    }

    /** {@inheritDoc} */
    @Nonnull
    public BookClientImpl build() {
        final BookClientImpl impl = populateBaseBuilder(new BookClientImpl());
        return impl;
    }
}
```

We will talk about `BookClientImpl` in the next chapter. But as you see, the implementation of the abstract methods is extremely simple. You will notice that we have used the Protobuf media type in the resource representation. This is because of the benefits it carries - lighter payload, fast processing etc. when compared to JSON.

***
__Extension Point: Custom endpoints__

In the previous chapter, we said that if your CRUD endpoints are simple, then you won't need any customization. But in case you need customization, then your builder will also have to be customized accordingly. For example, consider that you have a few `/tasks/*` endpoints or some hierarchical endpoints. Now these endpoints should be constructed set into the client implementation because the framework only sets the basic CRUD and Search endpoints. So you will have to create methods to construct these endpoints. If we take the same example of a library, then you will have to construct the endpoints for `#borrowBook` and `#returnBook`. Say the endpoints are:

* `/books/{bookId}/tasks/borrow`
* `/books/{bookId}/tasks/return`
 
You would implement the following methods in the builder:

```java
    /**
     * Creates and returns the borrow-book URI template.
     * 
     * @return Returns the borrow-book URI template.
     */
    @Nonnull
    protected UriTemplate getBorrowBookEndpoint() {
        try {
            return buildFromTemplate(getServiceBaseUrl()).literal(getResourceCollectionPath()).path(var("bookId"))
                    .literal("/tasks/borrow").build();
        } catch (final MalformedUriTemplateException e) {
            throw new IllegalStateException("Error creating borrow-book endpoint template", e);
        }
    }

    /**
     * Creates and returns the return-book URI template.
     * 
     * @return Returns the return-book URI template.
     */
    @Nonnull
    protected UriTemplate getReturnBookEndpoint() {
        try {
            return buildFromTemplate(getServiceBaseUrl()).literal(getResourceCollectionPath()).path(var("bookId"))
                    .literal("/tasks/return").build();
        } catch (final MalformedUriTemplateException e) {
            throw new IllegalStateException("Error creating return-book endpoint template", e);
        }
    }
```

Once you have these methods, you can then customize your `#build` method to set these additional endpoints into the client implementation. See code snippet below:
```java
    /** {@inheritDoc} */
    @Nonnull
    public BookClientImpl build() {
        final BookClientImpl impl = populateBaseBuilder(new BookClientImpl());
        impl.setBorrowBookUriTemplate(getBorrowBookEndpoint());
        impl.setReturnBookUriTemplate(getReturnBookEndpoint());
        return impl;
    }
```

For an explanation of impl#setBorrowBookUriTemplate and impl#setReturnBookUriTemplate, you will have to read the next chapter and specifically the Customzization section.

You can see an example of a custom client builder [here](http://git.covisintrnd.com/eng-iot-core/device-service/blob/master/client/src/main/java/com/covisint/platform/device/client/device/DeviceClientBuilder.java).

***

## How to use the Client Builder?

Explaining how to use the client builder will be incomplete before we actually explain the client implementation. So, go through the next chapter. And at the end of it, you'll see sample code on the entire Internal Client usage.

<table>
  <tr>
    <td><a href="client-interface">&lt; 4.1. Interface</a></td>
    <td align="right"><a href="client-impl">4.3. Implementation &gt;</a></td>
  </tr>
</table>