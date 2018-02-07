<table>
  <tr>
    <td><a href="solr">&lt; 6. Index & Query Engine with Solr</a></td>
    <td align="right"><a href="guava">8. Client-side Caching with Guava &gt;</a></td>
  </tr>
</table>
# 7. Server-side Caching with Redis

The server supports resource caching.  It automatically manages and/or uses the cache for all CRUD and search operations.  Caching is performed at the DAO layer, inside ```com.covisint.core.http.service.server.dao.BaseResourceRiakDao```.  There are a few reasons why it is implemented here as opposed to within the service layer, but that is outside the scope of this document.

The following DAO methods are cached:

| Method     | Description | Cache Key | Example |
| ---------- | ----------- | --------- | ------- |
| ```#count```    | Returns the count of resources of a particular resource type. | ```framework:$resourceClass:$realm:count``` | ```framework:com.app.Thing:ABC:count``` |
| ```#exists```   | Returns true if a resource exists for the given id, false otherwise. | ```framework:$resourceClass:$realm:$id?``` | ```framework:com.app.Thing:ABC:123?``` |
| ```#getById```  | Returns a resource by the given id. | ```framework:$resourceClass:$realm:$id``` | ```framework:com.app.Thing:ABC:123``` |

(Note that if the resource is not realm scoped, then the realm component of the cache key will not exist.)

When any of these DAO methods are executed and get the result from the datastore, they cache that result in the cache service, [Redis](http://redis.io/).

So these methods populate the cache.  On the other end, we have mutator methods that can change the stored state of a resource.  When this happens, the framework automatically recreates, purges or updates the cached value of the modified resource.  This ensures that the state of cached resources in Redis is almost real-time in sync with the state of the persisted resources.

Here are the framework DAO methods that mutate the cached values in Redis:

| Method     | Description | Cache Key |
| ---------- | ----------- | --------- |
| ```#add(resource)```    | Creates a new resource. | ```framework:$resourceClass:$realm:$resource.id``` |
| ```#update(resource)```   | Updates a given resource. | ```framework:$resourceClass:$realm:$resource.id``` |
| ```#delete(id)```  | Deletes a resource by id. | ```framework:$resourceClass:$realm:$id``` |
| ```#delete(resource)```  | Deletes a given resource. | ```framework:$resourceClass:$realm:$resource.id``` |

(Note that if the resource is not realm scoped, then the realm component of the cache key will not exist.)

The framework is actually not opinionated as to which cache implementation is used under the covers.  It leaves this to the applications to determine.  The framework simply uses the cache annotations present in ```spring-core``` on the cacheable methods:

*  ```@org.springframework.cache.annotation.Cacheable```: Indicates that a method has a cacheable return type.
*  ```@org.springframework.cache.annotation.CachePut```: Indicates that a method will put or replace a cache value based on the determined key.
*  ```@org.springframework.cache.annotation.CacheEvict```: Indicates that a method will remove a cache value based on the determined key.

## Configuration

The HTTP services use the [Jedis](https://github.com/xetorthio/jedis) client to communicate with the Redis cache service.  This is done by simply adding the following into the server module of each application:

```xml
  <dependency>
    <groupId>com.covisint.core.support.redis</groupId>
    <artifactId>redis-support</artifactId>
  </dependency>
```

This dependency will in turn bring in ```redis.clients:jedis``` and ```org.springframework.data:spring-data-redis```.  The latter will act as a bridge between Spring and Jedis, and will bring the cacheable annotations to life.

The cache is configured via ```spring-cache.xml```, which sets up the following:

1. The Jedis connection factory (for connecting to Redis)

```xml
  <bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
            p:host-name="${cache_service_host}" p:port="${cache_service_port:6379}" />
```

2. The Redis cache manager, which is a Spring ```org.springframework.cache.CacheManager```

```xml
  <bean id="cacheManager" class="org.springframework.data.redis.cache.RedisCacheManager" 
        c:template-ref="redisTemplate" />
```

3. A special cache resolver, used to resolve the appropriate cache for any given resource.  This is a type of Spring ```org.springframework.cache.interceptor.AbstractCacheResolver```

```xml
  <bean id="resourceTypeCacheResolver" class="com.covisint.core.support.redis.ResourceTypeCacheResolver" 
        c:cacheManager-ref="cacheManager" />
```

4. A Redis template that acts as a circuit breaker of sorts if and when the connection to Redis ever goes down.

```xml
  <bean id="redisTemplate" class="com.covisint.core.support.redis.SelfHealingRedisTemplate"
        p:connection-factory-ref="jedisConnectionFactory" p:reactivationDelayMillis="3000" />
```

Note: By default, the cache TTL is set to 0, which means that cache entries never expire.  Because the framework automatically maintains the cached entries on mutation ops, a never-expiring cache is safe and actually optimal because in theory every GET should result in a cache hit. 

These configurations and the spring-cache.xml in general are automatically included for you when using the [Maven archetype](quickstart) to initialize new projects.  The ```spring-cache.xml``` __must__ be included in every service to promote homogeneity, do not remove it unless you are sure you know what you are doing.

***
__Extension Point: Custom Caches__

Sometimes there are custom DAO methods needed which modify resources in a manner different from the framework's ```#update``` method.  For example, consider the following:

```java

  @CacheEvict(key = "#root.target.buildCacheKey(#resource.id)")
  public void activate(MyResource resource) {
    
    // Perform 'activation' here.  
    // Let's assume that this updates a certain flag on the passed resource.

  }
```

This custom DAO method is responsible for activating a resource, which ultimately changes the stored resource state.  In this scenario, cache must be managed manually and the cached resource (if it exists) will be evicted and subsequent requests for that resource will be forced to go to the persistence store for a full fetch.

Other scenarios call for a different cache key altogether, and the framework's id-driven cache keys are not sufficient.  Consider this custom method:

```java

  @Cacheable(key = "surname")
  public Person getBySurname(String surname) {

    Person person = ... // Get from persistence store.

    return person;

  }

```

This method performs a retrieval by surname, and at first glance seems like the perfect use case for a cache operation.  However the person resource is fetched via surname, not id, and so the framework CRUD operations aren't going to be useful here.  There is nothing stopping the developer from using the same caching infrastructure to perform custom caching like this.

***

***
__Extension Point: Cache Management__

There may be times when you need to specify a real TTL, instead of using the framework's default of 'infinity'.  This can be done on a type-by-type basis, which means that if your HTTP service defines resource A, B and C, you can adjust the TTL for A and B to have some value, and leave C assigned to never-expire cache.  It is done like this:

```xml
  <util:map id="expires">
      <description>Map of TTL settings, per resource type.  If no resource-level setting is specified, 
          then it falls back to whatever is set for RedisCacheManager.defaultExpiration</description>
      <entry key="com.app.Peanut" value="60" />
  </util:map>

  <bean id="cacheManager" class="org.springframework.data.redis.cache.RedisCacheManager" c:template-ref="redisTemplate"
       p:defaultExpiration="${cache_default_ttl_seconds:0}" p:expires-ref="expires" />
```

The configuration above will assign a TTL of 60 seconds only to resources of type ```com.app.Peanut```.

***

<table>
  <tr>
    <td><a href="solr">&lt; 6. Index & Query Engine with Solr</a></td>
    <td align="right"><a href="guava">8. Client-side Caching with Guava &gt;</a></td>
  </tr>
</table>