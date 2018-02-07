<table>
  <tr>
    <td><a href="implementation-client">&lt; 4. Client</a></td>
    <td align="right"><a href="client-builder">4.2. Builder &gt;</a></td>
  </tr>
</table>
# 4.1. Client Interface

The framework provides an interface called `ResourceClient`. It operates on resources. All clients which act on resources will have to implement this interface or a child of this interface. The interface provides definition of all CRUD APIs and Search API. Here is a description of  each:

1. `CheckedFuture<T, ServiceException> get(@Nonnull @NotEmpty String id, @Nonnull HttpContext httpContext)`: Gets a resource by its ID. This methods returns a future containing the requested resource, if it exists.

2. `CheckedFuture<T, ServiceException> get(@Nonnull T resource, @Nonnull HttpContext httpContext)`: Conditionally gets a resource based on some previously-fetched version of that resource. The previously-fetched resource should have a version set. This method returns a future containing either a) the current resource at the server, if newer than the passed resource, or b) return back the given resource if it is the most recent version available

3. `CheckedFuture<T, ServiceException> add(@Nonnull T resource, @Nonnull HttpContext httpContext)`: Adds a new resource. This methods returns a future containing the added resource.

4. `CheckedFuture<T, ServiceException> persist(@Nonnull T resource, @Nonnull HttpContext httpContext)`: Updates an existing resource. This methods returns a future containing the updated resource. Note that this only works on existing resources. If you want to add a new resource then use the `#add` method

5. `CheckedFuture<T, ServiceException> delete(@Nonnull @NotEmpty String id, @Nonnull HttpContext httpContext)`: Deletes an existing resource using the ID provided. This method returns the deleted resource, if it had existed.

6. `CheckedFuture<List<T>, ServiceException> search(@Nonnull Multimap<String, String> searchCriteria,`
   `@Nonnull SortCriteria sortCriteria, @Nonnull Page page, @Nonnull HttpContext httpContext)`: This method is meant for the Search API.  This method returns a future containing the list of all matching resources found. Definition of the method arguments is explained in the below sections. 

7. `CheckedFuture<List<T>, ServiceException> search(@Nonnull Multimap<String, String> searchCriteria,`
   `@Nonnull SortCriteria sortCriteria, @Nonnull Page page, @Nonnull HttpContext httpContext,`
   `@Nonnull MediaType mediaType, @Nonnull @NotEmpty String searchResourcesEndPoint)`: This is similar to the method described above for the Search API. But with a slightly different flavor. there may be times, when your search endpoint is slightly different from what the framework automatically constructs. So, you can provide the base URI of your search endpoint here along with the content type media type of the results. In most cases, you would not need this method. The previous one is very well sufficient.

## HTTP Context

In all of the above method definitions, you will see that there is a method argument which takes the HTTP Context. This is nothing but the context which holds the HTTP session data. For the microservices built using the HTTP Service Framework to work, at the least, you need to set the following information in the HTTP Context:
* Realm
* Requestor ID
* Requestor App

The below code snippet shows you how to set this. You can see more details of this in the [Client Implementation](client-impl) chapter:

```java
final HttpContext context = new BasicHttpContext();
context.setAttribute(XRealmInterceptor.CONTEXT_ATTRIBUTE_NAME, "ABC");
context.setAttribute(XRequestorInterceptor.CONTEXT_ATTRIBUTE_NAME, "Sam");
context.setAttribute(XRequestorApplicationInterceptor.CONTEXT_ATTRIBUTE_NAME, "Sam's App");
```

## Search API Method Arguments

Here is a description of the various Search API arguments and how they can be used:

* `searchCriteria`: This defines the parameters to search by. This is a `Multimap`. So when you have to provide your search parameters, you have to provide the search field name as the key of this `MultiMap`. The values for this will be one or more values to search for, against the search field i.e., In case you want to search for books by their isbn, then you can set the search field values as follows:

  ```java
  final Multimap<String, String> searchCriteria = HashMultimap.create();
  searchCriteria.put("isbn", "1-56619-909-3");
  searchCriteria.put("isbn", "0-68416-326-8");
  ```

  _NOTE: The search criteria will work *only* on fields that are marked with the @Searchable annotation. In case you give any other parameter, then it will simply be ignored by the framework._
 
* `sortCriteria`: The order in which the results should be sorted. You can provide sort criteria for different fields and mention if they are should be ascending (+) or descending (-) as follows:
        
  ```java
  SortCriteria stepSortCriteria = SortCriteria.builder().parseSortField("-creation").parseSortField("+title")
          .parseSortField("author").build();
  ```

  The above code snippet helps you sort the results descending by `creation`, ascendinig by `title` and ascending by `author`. You will see that providing + or nothing will mean it is ascending order. The first sort field provided will the be primary sort field. In the above example, it is the `creation` timestamp is the primary sort field. The others will be secondary sort fields in the order provided.
 
  _NOTE: The sort criteria will work *only* on fields that are marked with the @Sortable annotation. In case you give any other parameter, then it will simply be ignored by the framework._

* `page`: This defines how many results should be retrieved per page and which page should be retrieved. This is very useful when you have a large amount of results to be returned and you want to retrieve them in smaller chunks. In case you want to retrieve the 10th page, where each page is supposed to contain a maximum of 100 records, then this is how you will define the page parameter (i.e., provided you have that much data to display)
    
  
  ```java
  Page page = new Page(10, 100);
  ```

  _NOTE: By default, the framework gives you the results of 1st page, with each page holding 50 records. The maximum number of results you can retrieve per page is 200._

## Return CheckedFuture

In each of the above method definitions, you will see that the methods return a [CheckedFuture](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/util/concurrent/CheckedFuture.html) either containing the resource or the list of matching resources. This is because the base client implementation executes all the requests asynchronously. To retrieve the actual resource, you just have to call the `#checkedGet`. More details of this in the [Client Implementation](client-impl) chapter 


## Your Client Interface

For your resource, you will have a client interface of your own, which extends the `ResourceClient`. You do not have to define anything additional here, in case your APIs are basic CRUD operations, your URI paths are not hierarchical in nature and you do not have any special media type parameters. But if your requirements are slightly more than basic CRUD, then define your method definitions in your interface.

A simple client interface for your resource will look like this:

```java
public interface BookClient extends ResourceClient<Book> {

}
```

***
__Extension Point: Custom endpoints__

If you want to see a more complex and customized Client Interface, then see [this](http://git.covisintrnd.com/eng-iot-core/device-service/blob/master/client/src/main/java/com/covisint/platform/device/client/device/DeviceClient.java).

As a guideline, you will have to create a method definition for each custom endpoint. For example, in case you were using the book resources in a library application, then you could have custom endpoints for borrowing and returning. So you will have method definitions like: `#borrowBook`, `#returnBook` etc.

***

<table>
  <tr>
    <td><a href="implementation-client">&lt; 4. Client</a></td>
    <td align="right"><a href="client-builder">4.2. Builder &gt;</a></td>
  </tr>
</table>