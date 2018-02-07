<table>
  <tr>
    <td><a href="riak-converter">&lt; 2.5. Riak Convertor</a></td>
    <td align="right"><a href="implementation-webapp">3. Webapp &gt;</a></td>
  </tr>
</table>
# 2.6. Riak Indexer

There may be times when you need bells and whistles for your CRUD endpoints. For ex:

* You may want to update a mapping table for entity relationships whenever entities in that relationship are added / updated / deleted
* You may want to create some reverse index tables which help in faster lookup and so on

The framework gives the flexibility of adding such additional features easily, in the form of Riak Indexers. 

## **Interface**

The framework provides an interface called `RiakIndexer` which has to be implemented by all Riak indexers. Here is what each method is meant for:

1. `#getResourceType`: Gets the type of resource being operated upon

2. `#getNamespace`: Gets the Riak namespace in which the indexer should operate

3. `#getIndexKey`: Gets the Riak Key for the indexer

4. `#getIndexValues`: Gets the values that should be persisted by the indexer. By default this will be a `Set<String>`

5. `#onAdd`: This method is an extension point to the DAO's `#add` method. Additional process for `#add` is done in this method.
 
6. `#onUpdate`: This method is an extension point to the DAO's `#update` method. Additional process for `#update` is done in this method.
  
7. `#onDelete`: This method is an extension point to the DAO's `#delete` method. Additional process for `#delete` is done in this method.

8. `#get`: This method gets the values stored by the indexer. Again, the default values that are retrieved are a `Set<String>`


## **Abstract Indexer**

Now that we've understood the `RiakIndexer` interface, we can go ahead and create an implementation for all the methods. But just like Riak converters, we have 2 main abstract implementation classes for the `RiakIndexer`, as part of the `riak-support` project: 

* `AbstractResourceRiakIndexer`: This is the main base implementation class. It provides default implementation for all the CRUD extension methods of the `RiakIndexer`. i.e., `#onAdd`, `#onUpdate`, `#onDelete` and `#get`. When we say default implemenation, we mean that these provide implementation to create a RiakSet using the `#getIndexKey` and `#getIndexValues`. This can be used to create a relationship bucket. If you need other types of indexing, then you will have to implement them on your own
 
* `AbstractRealmScopedResourceRiakIndexer`: This is just an extension of `AbstractResourceRiakIndexer` for realm scoped resources.

**Only** the following methods will have to implemented in your indexer if you extend one of the above abstract indexers and the default implementation works for you:

* `#getResourceType`
* `#getNamespace`
* `#getIndexKey`
* `#getIndexValues`

## **Your Riak Indexer**

Every resource can have zero or more indexers. You can extend your indexer from `AbstractResourceRiakIndexer` or `AbstractRealmScopedResourceRiakIndexer` for the default implementation.

You will have to configure the indexers with the DAO in your Spring context XML (spring-persistence.xml) as follows. 

_NOTE: Coniguration of riakClient and bookConverter is not shown here_


    <bean id="bookIndexer1" class="com.covisint.platform.book.server.book.BookRiakIndexer1" />

    <bean id="bookIndexer2" class="com.covisint.platform.book.server.book.BookRiakIndexer2" />

    <bean id="bookDao" class="com.covisint.platform.book.server.book.BookRiakDao"
        c:client-ref="riakClient" c:converter-ref="bookConverter">
        <property name="riakIndexers">
            <list>
                <ref bean="bookIndexer1" />
                <ref bean="bookIndexer2" />
                <!-- ... and so on --> 
            </list>
        </property>
    </bean>
 
You can see an example of a Riak indexer class in the group-service microservice.

<table>
  <tr>
    <td><a href="riak-converter">&lt; 2.5. Riak Convertor</a></td>
    <td align="right"><a href="implementation-webapp">3. Webapp &gt;</a></td>
  </tr>
</table>