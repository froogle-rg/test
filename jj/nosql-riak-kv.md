<table>
  <tr>
    <td><a href="implementation-nuts-bolts">&lt; V Implementation: The Nuts and Bolts</a></td>
    <td align="right"><a href="solr">6. Index & Query Engine with Solr &gt;</a></td>
  </tr>
</table>
# 5. NoSQL Persistence with Riak KV

## What is Riak KV?

Riak KV is a distributed key/value NoSQL datastore. It is a product from [Basho Technologies](http://basho.com/). A key/value datastore is the simplest of all NoSQL datastores. It is schema-less, i.e., it has no predefined schema or data definition model that the data  being persisted will have to confirm to. Simply put, it is like a dictionary of key and value pairs. 

The value being persisted can be a simple string, JSON document, binary value, BLOB, etc. Because of the schema-less nature of KV stores, it is easy to store any type of data without having to think much about confirming to a specific data type / schema. It is also easy to accommodate requirement changes to accommodate newer versions of resources to be stored. Consider a scenario where a requirement change comes in and you have to now store additional properties for a particular resource. If it were a datastore having a schema, we would have to make changes to the schema, ensure default values are persisted for the new properties of existing data and ensure that nothing breaks for existing users. But with KV stores, we don't have to worry about anything. We can simple start persisting the additional data, without any configuration changes! KV datastore magically takes care of things for you.

## Why Riak KV? The background and history 

Before we go into the reason for choosing Riak KV over other datastores, let us first understand what was the requirement we had to select a datastore that would fit into the HTTP Service Framework. Mentioned below are some of the important requirements we had:

* Ability to do CRUD operations on the resources very efficiently
* Ability to search for resources using different attributes (i.e., searchable fields). Search should also support pagination, case insensitive searches, combination of search fields, sorting etc.
* Easy integration with a Search / Indexing engine for search capabilities
* High performance on both reads and writes, even with massive amounts of data
* And we needed all the buzz associated with any good application - highly scalable, consistency, availability, low response time etc.

Well, these seem like requirements that any normal application would have when they would like to choose a datastore. And it also seems like these requirements can be met by any good datastore available. Below is an explanation of why we finally chose Riak KV over any other datastores.

### **RDBMS** 

The primary reason to create the HTTP Service Framework 2.0.0 was to move away from the traditional RDBMS datastore. Why? Because it simply did not meet the performance and scalability requirements of the ever increasing data load. So we invariably had to give up the good old RDBMS. 

### **MongoDB (Document-based NoSQL datastore)** 

The most popular document-based NoSQL datastore [MongoDB](https://www.mongodb.org/) was evaluated. The document-based datastores are similar to KV datastores, but the values are stored as 'documents' - JSON, XML, BSON etc. and hence there is some form of structure and encoding associated with the value stored. These datastores are flexible to store complex data structures. It can store documents hierarchically i.e., documents within documents. The metadata of the values stored is also embedded with the data stored. Hence searching data based on any attribute is extremely easy. 

Despite all the powerful features of document-based datastores, it came with its own set of issues. Some of them are listed here: 

* Memory usage is higher compared to other datastores since it stores the metadata associated with the each document. The metadata was a boon and a bane! 
* Schema design is extremely important in MongoDB and changing schemas over a period of time may not be a trivial job. 
* Read performance degrades when the documents are nested too deep. 
* Although MongoDB claims to be eventually consistent, there are known issues in consistency. 

### **Cassandra (Column-based NoSQL datastore)** 

[Cassandra](http://cassandra.apache.org/) is a column-based NoSQL datastore from Apache. We evaluated the most popular Cassandra distribution which is provided by [DataStax](http://www.datastax.com/). A column-based NoSQL datastore stores the data in columns as opposed to rows in RDBMS. A group of columns is logically grouped into column families. For a simple explanation, a column family is like a 2-dimensional array. A column family contains many rows and each row contains many columns. One row in a column family can have different columns than that of another row of the same column family. This is essentially useful to store related data together. For example: An employee and employee's address can be 2 different rows, but belonging to the same column family. The columns in employee and employeeAddress can be very different. You can read more on Cassandra [here](http://www.planetcassandra.org/what-is-apache-cassandra/).

### **Cassandra** versus **Riak KV**

Cassandra seemed like a strong contender for Riak KV in all the requirements that we saw above for the HTTP Service Framework. It was now time for us to get into more specifics. One of the key requirements for us was **Search** and neither Cassandra nor Riak KV fulfilled the search requirements that we had. Neither of them came with search capabilities out-of-the-box. Both of them only allowed searching by the 'key' of the data. If we had to search by any other field, then we had to introduce Secondary Indexes. But this would degrade performance as data grew. So we now had to choose a Search engine which would supplement the data storage capabilities of each datastore. The following different combinations  of search engines were used for evaluation:

* **DataStax Cassandra + ElasticSearch** - [ElasticSearch](https://www.elastic.co/) is an indexing engine which has a rich feature set and client libraries in many languages. Using ElasticSearch for the search requirements of the HTTP Service Framework seemed apt as it catered to all the use-cases we have. The performance statistics of using ElasticSearch for Search with Cassandra as the datastore were impressive. Integrating ElasticSearch with Cassandra meant syncing data from Cassandra to ES via code / via a background process / using a plugin. But we did not go ahead with ElasticSearch. Because, at the time of performing this evaluation, ElasticSearch did not have a built in capability for cross-DC replication. This could really become a pain-point when we scale out across multiple DataCentres. Apart from this, there is also a known [split-brain](https://aphyr.com/posts/323-call-me-maybe-elasticsearch-1-5-0) problem in ElasticSearch. And hence ElasticSearch was dropped. 

* **DataStax Cassandra + Stratio's Cassandra Lucene Index** - [Stratio's Cassandra Lucene Index](https://github.com/Stratio/cassandra-lucene-index) is a plugin to Cassandra. It provides many search capabilities similar to ElasticSearch or Solr. This is based on the implementation of Cassandra's Secondary indexes. Since this is a plugin to Cassandra, there is no need to 'sync' data between Cassandra and any other system. But the performance evaluation showed very poor results as compared to ElasticSearch evaluation done previously. Apart from this, there were quite a few [features not supported](https://github.com/Stratio/cassandra-lucene-index#features) at the time of performing this evaluation. Hence we needed to find another indexing / search engine.

* **DataStax Enterprise Cassandra (Bundled with Solr)** versus **Riak Enterprise (Bundled with Solr)** - These were the final 2 contenders among the search engines and this also was a key factor to decide between Riak KV and Cassandra. Both Enterprise Editions came bundled with the powerful Solr indexing engine - [DataStax Enterprise Search](http://docs.datastax.com/en/datastax_enterprise/4.5/datastax_enterprise/srch/srchTOC.html) and [Riak Search](http://docs.basho.com/riak/latest/dev/using/search/). Both of these had similar features supported since both used Apache Solr under the wraps. Both took care of syncing data from the primary datastore (Cassandra / Riak KV) to the Solr nodes automatically. So the decision had to be made and it was made based on running some performance metrics for the search use cases on both the stacks on exact same infrastructure setup. Cassandra and Riak KV were head-to-head initially. But as data grew, Riak KV came out as a clear winner even when data volume grew in leaps and bounds.


## Data Modeling with Riak

Now let's learn some basics on data modeling with Riak. We'll keep comparing things with RDBMS data modeling for a better explanation.

At a very high level, in RDBMS, we have the following main terminologies:

* **Schema**: A logical grouping of several database objects. 
* **Table**: Each schema has many tables. Tables store the data about each resource. Data for a single resource can be normalized and stored in multiple tables. 
* **Column**: Every attribute of a resource is stored in a separate column. Each column is associated with a datatype and length.
* **Row**: Every row holds the values for different instances of the resource. Each row will have the same columns always.
* **Constraints**: You can define constraints on each column so that data abides to certain rules. For ex: Not Null, Primary Key, Foreign Key, Unique value, etc.
* **Primary Key**: This can be a simple or composite key. This is the unique identifier of a particular row in a table.
* **Foreign Key**: Relation between resources is maintained using Foreign Key constraints.

In Riak, we have the following concepts:

* **[Bucket Type](http://docs.basho.com/riak/latest/dev/advanced/bucket-types/)** - A logical group of different Riak Buckets. You can define configuration details here which is inherited by all buckets. This can be somewhat compared to an RDBMS schema, but not entirely.
* **[Bucket](http://docs.basho.com/riak/latest/theory/concepts/Buckets/)** - This where the data i.e., Riak objects will be stored. This is analogous to an RDBMS table. In the HTTP Service Framework, data for realm scoped resources is stored in separate buckets per realm. This is done auto-magically by the framework, and you don't have to worry much. You just need to know the bucket name you are dealing with. For ex: Book resources will be stored in bucket called `book_service`. You just have to know this. But books in realmA and books in realmB are actually stored in buckets `book_service_realmA` and `book_service_realmB`
* **[Key](https://docs.basho.com/riak/1.1.4/references/appendices/concepts/Keys-and-Objects/#Keys)** - This is used to identify the object stored in Riak. So this should be a unique value. This is analogous to the RDBMS Primary key. But this may / may not be part of the object itself, unlike in RDBMS, where the primary key is part of the data. In the HTTP Service Framework, the resource id will be the key.
* **[Object](https://docs.basho.com/riak/1.1.4/references/appendices/concepts/Keys-and-Objects/#Objects)** - Data stored in Riak are objects. Each object is stored against a Key. This is analogous to an RDBMS row, but is more loosely defined and does not have columns and datatypes. In the HTTP Service Framework, the resource will be the object that is stored.
* **Constraints** - Unlike in RDBMS, Riak doesn't impose any constraints on the data being stored. In case you need something unique / non null etc. then you will have to manage this in your Entity Validators. 
* **Relation** - Unlike in RDBMS, Riak doesn't have a way to define relation between resources. But you can manage this be storing the data accordingly. Consider that you have 2 resources: `Book` and `Author`.
    * **1:1 relation**: Every Book has 1 Author and every Author has written only 1 book. You can store the `author.id` in the Book resource and / or you can store the `book.id` in the Author resource. When you retrieve the Book resource, you know the `author.id`, which is the Riak key to the Author resource. So you can go and fetch the Author resource from its bucket. Similarly retrieve Book using `book.id`
    * **1:n relation**: Every Book has 1 Author, but every Author has written many books. Storing the Book resource is simple and just as explained above in the 1:1 relation. Book resource also stores the author.id. But the Author resource will have to store many `book.ids`. In RDBMS, we will have to store this in separate columns / normalize and store in separate table. But with Riak, we have the flexibility of storing **all** the `book.ids` at once using *RiakSet*. So we store the set of `book.ids` along with Author resource. i.e.,  Author will have a field like: _"books" --> RiakSet_. Once you retrieve this RiakSet, you can then individually get all the Book resources from its bucket using these ids.
    * **n:m relation**: Every Book has many Authors. Each Author has written many Books. Here again, you will store the set of ids of the related resource in each resource, as explained above in 1:n relation, to store 'n' side of the relation.

    _Note: In case, you want to store the relation in a separate Riak bucket, for example, you want to store the book and author ids in a 'normalized' bucket, then you can use Riak Indexers as explained above._

## Riak data type and Java data type

In Riak, we have the following data types. Let's see which Java data types need to be stored as which Riak data type.

1. **RiakRegister** - This holds any `String` or binary value. This suits most of the simple attributes in the resource - Java primitive types and wrapper classes
2. **RiakCounter** - This is used for a sequence / counter attribute.
3. **RiakSet** - Similar to Java `Set`s, these store unique values. These can store  only `RiakRegister`s as their elements. You can store Java Sets here which hold elements of Java primitive data type / wrapper classes. i.e., `Set<String>`, `Set<Integer>`, etc.
4. **RiakMap** - Similar to Java `Map`s, these store key-value pairs. The key is a `String` or binary value and the value can be a `String` / binary value / another Riak data type, including `RiakMap` itself. This is the richest of all Riak data types. Every element in the map need not be of the same data type. This means that you can store a `RiakMap` against key1, whereas `RiakSet` against key2 and `RiakRegister` against key3.

Now you may wonder how to store other Java data types like `List`, nested classes etc. You have to ensure to fit the data into one of the available Riak data types. You will have to understand your requirement and then design how to store and retrieve your data. You will also have to consider Solr when you want to store your data which has searchable fields. More details on Solr can be found in the sections below. Some hints to store the different Java data types are:

1. **List** - You can store this in a RiakSet, if your data is always unique. But Lists are not meant to store uniqueness. And they are also ordered. You have many options to achieve this
    * Store the list as a `RiakMap` where the key of the map is the list index 
    * Store the list as a `RiakSet`, but prefix the list index to the value being stored. Something like "listIndex:listValue"
    

2. **Nested classes** - Depending on your requirement, this also can be stored as a `RiakMap` / `RiakSet` with some amount of custom coding to make them confirm to your requirements. Alternatively, you can also store them in independent buckets and have a way to link your main resource to these. Something similar to 1:n relation as explained above


## Gotchas!

And yes, there are gotchas!

* **_“Do not wait for someone else to validate your existence; it is your own responsibility” - Jasz Gill_**: Before you think it is philosophical, let us tell you that it is only resources we are talking about here :) As described above, Riak doesn't enforce any schema / datatypes on the data it stores. So you can't expect Riak to throw something like SQLExceptions when there is missing data / invalid data. This will have to be carefully handled in your Entity Validators and Riak should just be considered as a storage of 'cleansed' data.
* **_"Too much of anything is bad, but too much good whiskey is barely enough." - Mark Twain_**: But unfortunately, we are talking about data here ;) At the time of writing this document, Basho says that Riak does not perform well, if we store too much data in RiakMap or RiakSet. Too much they say is 1000+. But Basho plans to fix this soon. So watch out!
* **_“Let us never negotiate out of fear. But let us never fear to negotiate." - John F. Kennedy_**: Riak and Solr have to work together. So you will have to make some negotiations while modeling your data, so that data can be retrieved from Solr correctly later. This is especially important for all the `@Searchable` attributes of your resource. The framework already takes care of `Java Set`, `Java Map` and any other framework resources like `InternationString`, `ResourceReference` etc. But if you have complex data types / hierarchical objects, then you will have to first understand how to store this so that Solr can retrieve this later. Sometimes, you may have to split the data and store in a separate bucket. At other times, you might want to store this is as a concatenated String. It depends on a case-by-case basis. Once you decide the format, construct your Riak Converter accordingly.

## Configuration

Installing and setting up Riak and enabling Solr is out of scope of this document. Please refer to [Basho documentation](http://docs.basho.com/riak/latest/) for the same. Assuming that you have already setup your Riak Cluster with Solr enabled, here are the configurations you will have to do for your microservice to start working. Run these commands on the host where Riak is installed:

	# Create the Solr index for the microservice
	curl -XPUT $RIAK_HOST/search/index/book_service_idx -H'content-type:application/json' -d'{"schema":"microservice_default_schema"}'

 _NOTE: Assumption is that the Solr Custom Schema `microservice_default_schema` is already setup. See next section for more details._

    # Setup the Riak bucket-type for 'map' datatype and the previously created Solr index
    sudo $RIAK_BIN/riak-admin bucket-type create book_service_type '{"props": {"datatype":"map","search_index":"book_service_idx"}}'

    # Now activate the bucket-type
    sudo $RIAK_BIN/riak-admin bucket-type activate book_service_type

All your bucket-types *should* be of `map` datatype. As a thumb rule, create only one bucket-type per microservice. But use separate buckets for each resource. For ex: `Book` will belong to `book` bucket (which will then by dynamically suffixed by realm if it is a realm scoped resource) and `Author` will belong to `author` bucket. But both these buckets will be of type `book_service_type`. This should be sufficient for most parts. In case you wish to create different types, then configure your bucket types accordingly.

If you wish to create additional bucket types, for example, you want to create `set` bucket types for Riak Indexers, then you can create use the following configuration as well. This is _optional_

	# Setup the Riak bucket-type for 'set' datatype. Note that there is no Solr index for this, since it is not needed.
	sudo $RIAK_BIN/riak-admin bucket-type create book_service_set_type '{"props": {"datatype":"set"}}'

	# Now activate the bucket-type
	sudo $RIAK_BIN/riak-admin bucket-type activate book_service_set_type

<table>
  <tr>
    <td><a href="implementation-nuts-bolts">&lt; V Implementation: The Nuts and Bolts</a></td>
    <td align="right"><a href="solr">6. Index & Query Engine with Solr &gt;</a></td>
  </tr>
</table>