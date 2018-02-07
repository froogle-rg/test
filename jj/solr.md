<table>
  <tr>
    <td><a href="nosql-riak-kv">&lt; 5. NoSQL Persistence with Riak KV</a></td>
    <td align="right"><a href="redis">7. Server-side Caching with Redis &gt;</a></td>
  </tr>
</table>
# 6. Index & Query Engine with Solr

## What is Solr?
We've already mentioned Solr many times in the last few sections. So, what exactly is it? [Solr](http://lucene.apache.org/solr/) is an open source enterprise search platform developed by Apache. This is built over [Apache Lucene](https://lucene.apache.org/) which is a rich and powerful full-text search library developed in Java. Riak comes bundled with Solr and that is what is the base of our search platform.

In the above sections which explain to you how we chose Riak, we've already explained the need for a search and indexing engine. We need to support a rich set of search requirements in our microservices. And Solr helps with all that and provides a lot of other rich features. In the next section we will see what benefits does Solr bring in. 

## Benefits of Solr

You can refer to [Apache Solr Features](http://lucene.apache.org/solr/features.html) to understand the exhaustive features and benefits that Solr brings in. Listing a few here:

* Advanced full-text search including wildcard search, phrase search, token search, fuzzy search and a lot more! 
* Simple and intuitive query DSL 
* Near real time indexing
* Proven to handle large scales of data
* Comprehensive Administration interface and many more

## HTTP Service Framework and Solr

In the previous section about configuring Riak, we mentioned that you have to setup Riak with Solr enabled. By default, Solr is not enabled in Riak. So you have to explicitly enable it and do bucket type configurations as mentioned above. Once this is done, all the data that you persist in Riak will automatically start flowing down to Solr as well. And all the data will now start getting indexed in Solr. By indexing we mean that data is now getting ready to be searched.

### **Solr field names**

Since we are dealing with Riak Data types, all the data that is persisted in Solr will be suffixed by the Riak data type. For example, consider the following   attributes and their Solr names:

| Attribute   | Java data type        | Riak Data type | Solr field name  |
| ---------   | -------------         | -------------  | ---------------  |
| id          | `Long`                | `RiakRegister` | id_register      |
| version     | `String`              | `RiakRegister` | version_register |
| description | `InternationalString` | `RiakSet`      | description_set  |
| author      | `ResourceReference`   | `RiakSet`      | author_set       |
| names       | `Map<String, String>` | `RiakSet`      | names_set        |

As you can notice, there is a suffix for each field which depicts the Riak Data type of the field. So when you use the extension points in the DAO for search field name, remember these suffixes.

### **RiakMap and Solr**

You may notice in the previous section that a Java `Map<String, String>` was stored as a `RiakSet`. Why didn't we instead use a `RiakMap` when it is readily available? This is specifically needed when the field is a `@Searchable` field. Let us explain with an example.

Consider that `names` is an attribute of a resource which is a `Map<String, String>`. It holds the language and name of the resource. If we store this in Riak as a RiakMap, then Solr also stores it as a map, but here the name of the Solr field will now become dynamic:

```java
        final Map<String, String> names = new HashMap<String, String>
        names.put("en", "Name in English");
        names.put("fr", "Name in French");
```

Solr field name will now become: `names_en` and `names_fr`. And these fields will hold the values. This will work fine as long as `names` is not a `@Searchable` field. But in case this is a searchable field, then we won't be able to search for names across different languages. Hence we will instead store such Java `Map`s as `RiakSet`s. We will store the Map's key and value into a `RiakSet` as key:value concatenated `String`. With this, we can use Solr's wildcard search to search for names across all languages. 

## Additional interfaces of Solr

While HTTP Service Framework automatically plugs in Riak Solr and gives you extension points, there are times when you may need to just query Solr independently during your development or debugging. Or you may need to administer / monitor Solr. Here are a few interfaces that come out-of-the-box.

### **Solr Administrative User Interface**

By default, you can access the Solr Admin UI at: `http://{hostname}:8093/internal_solr`. The default port of Solr is 8093. But this can be change by specifying the `search.solr.port` setting in each node. This Admin UI is not Riak specific. This is the default that comes with Apache Solr. Admin UI helps administrators and developers to view configuration, run queries on the different Solr indexes created and analyze fields. An exhaustive documentation on the Admin UI is available in the [Apache Solr Wiki](https://cwiki.apache.org/confluence/display/solr/Overview+of+the+Solr+Admin+UI)

### **Solr Query interface using Solr's Port**

Apart from using the Admin UI for querying, you can also directly fire queries on Solr using the URL `http://{hostname}:8093/internal_solr/{index}/select?q=*%3A*` where index is the search index you have created previously. For ex: book_service_idx. This will by default give all the data in Solr for the index. You can add more filtering and narrow down your search with query parameters. You can find the list of query parameters supported by Solr [here](https://cwiki.apache.org/confluence/display/solr/Query+Screen)

_NOTE: URL is encoded here_

### **Solr Query interface using Riak's HTTP Port**

Additionally, you can also search in Solr using the the Riak HTTP Port. The default port is 8098. URL is `http://{hostname}:8098/search/query/{index}?q=*%3A*`. The same query parameters will apply as described above for querying on Solr's port

_NOTE: URL is encoded here_

### **Other Solr Query interfaces**

You can also query Solr with many other interfaces like Command Line, Erlang, Curl etc. For a detailed list of the different interfaces, refer [this](https://docs.basho.com/riak/1.3.2/cookbooks/Riak-Search---Querying/).

## Configuration

Solr comes with a default schema that will suffice for most requirements. But the HTTP Service framework has a custom Solr schema that is required for the Search use cases to work. Hence, set this up first.

	# First, copy the 'microservice_default_schema.xml' into the Riak host.
    # This is a custom Solr schema needed for the HTTP Service Framework's search functionality to work

	# Next, create the custom Solr schema. This only a one-time setup. All microservices can make use of this.
	curl -XPUT -d @microservice_default_schema.xml -H 'Content-Type: application/xml' $RIAK_HOST/search/schema/microservice_default_schema

In case you need a schema with more customization, then you can create a new schema. Further reading on custom schema creation [here](http://docs.basho.com/riak/latest/dev/advanced/search-schema/#Creating-a-Custom-Schema) 

<table>
  <tr>
    <td><a href="nosql-riak-kv">&lt; 5. NoSQL Persistence with Riak KV</a></td>
    <td align="right"><a href="redis">7. Server-side Caching with Redis &gt;</a></td>
  </tr>
</table>