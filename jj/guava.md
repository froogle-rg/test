<table>
  <tr>
    <td><a href="redis">&lt; 7. Server-side Caching with Redis</a></td>
    <td align="right"><a href="dropwizard-metrics">9. Server Metrics with Dropwizard &gt;</a></td>
  </tr>
</table>
# 8. Client-side Caching with Guava

The clients support in-process caches for GET responses.  This applies only to get-by-id operations, not searches.  Search responses are not cached due to a variety of reasons and is out of the scope of this document.  

Note: The resource cache is keyed by the resource id, which guarantees that it is unique among other cached objects for that resource type.

To enable client side caching you will need to tell the client builder what your cache parameters are.  This is done with a ```com.covisint.core.http.service.client.CacheSpec``` as shown here:

```java

  import com.covisint.core.http.service.client.CacheSpec;
  import com.covisint.core.http.service.client.CacheSpec.ExpirationMode;

  ...
  // Build a new CacheSpec, setting the TTL to 2500 milliseconds after the entry is written, 
  // and limiting the max number of entries to 100.
  final CacheSpec spec = CacheSpec.builder().expiration(2500, ExpirationMode.AFTER_WRITE).maxElements(100);

  ...
  
  // Set the cache spec into the builder as you are building the client.
  myClientBuilder.setCacheSpec(spec);

  ...

```

This example will employ a LRU cache inside the client.  An important caveat here is the possibility of stale cache.  If a resource is changed on the server after the client fetches and caches it, then the consuming application will not see the fresh resource state until the TTL expires.  Therefore, make sure that your application can tolerate this scenario and set the TTL appropriately.  By default, client caching is disabled and GET operations will always make an HTTP request to the server.

Read more on [Guava Caches](https://code.google.com/p/guava-libraries/wiki/CachesExplained).

***
__Extension Point__

You client will need to expose non-CRUD APIs as custom Java methods.  For any custom GET APIs that your client supports, you can choose to cache their responses too if needed.  This will work under the same constraints as the caching offered by the framework - the cache is governed by a TTL and max items (or max weight).  You will have to write the cache maintenance logic in your custom method manually, like so:

```java
    import com.google.common.util.concurrent.CheckedFuture;
    import com.google.common.util.concurrent.FutureCallback;
    import com.google.common.util.concurrent.Futures;

    ....

    // Some code omitted for brevity.
    CheckedFuture<MyResource, ServiceException> getSomething(HttpContext httpContext) {

        // Build the cache key.
        String key = String.format("my-cache-key", id);
        
        // Attempt to fetch value from cache.
        MyResource cached = (MyResource) getLocalCache().getIfPresent(key);

        if (cached != null) {
            // Cache value found.  Return it.
            return Futures.immediateCheckedFuture(cached);
        }

        CheckedFuture<MyResource, ServiceException> result = ... // Make service call.

        // Add future callback that will put the result in cache when it returns.
        Futures.addCallback(result, new FutureCallback<MyResource>() {

            public void onSuccess(MyResource resource) {
                // Put the result into the cache.
                getLocalCache().put(key, resource);
            }

            public void onFailure(Throwable t) {

            }

        });

        return result;
    }
```

This method handles both the service call as well as managing (and retrieving from) local cache.  The astute reader may ask "What happens if caching is disabled?".  In this case, ```getLocalCache()``` will return a Guava cache that has a TTL of 0, which effectively disables caching as it purges the entry as soon as it is put in.

***

## Configuration

See javadoc for ```com.covisint.core.http.service.client.CacheSpec``` for full details.  A cache spec object can be set up with:

1. TTL
2. Either a max elements limit OR a max weight limit.
   * Max elements: The maximum number of entries the local cache can support before it starts evicting older ones to be able to accept new additions.
   * Max weight: The maximum weight of entries the local cache can support.  Weight is determined using a ```com.google.common.cache.Weigher```:

```java
  Weigher<String, Library> weigher = new Weigher<String, Library>() {
            
    public int weigh(String key, Library value) {
      return value.getBooks().size() + value.getMagazines().size();
    }
            
  };
        
  CacheSpec spec = CacheSpec.builder().maxWeight(2000, weigher).build();

  ...
```

In the example code snippet above, we are telling the cache spec to limit all cached libraries so that the sum of all of their books and magazines never exceed 2000.

<table>
  <tr>
    <td><a href="redis">&lt; 7. Server-side Caching with Redis</a></td>
    <td align="right"><a href="dropwizard-metrics">9. Server Metrics with Dropwizard &gt;</a></td>
  </tr>
</table>