<table>
  <tr>
    <td><a href="dao">&lt; 2.4. Data Access Object</a></td>
    <td align="right"><a href="riak-indexer">2.6. Riak Indexer &gt;</a></td>
  </tr>
</table>
# 2.5. Riak Convertor 

Every resource before being persisted into the Riak store, will need to be converted to a specific format which Riak understands. The Riak Converter does just this. It converts the resources to [Riak Data Types](https://docs.basho.com/riak/2.0.1/dev/using/data-types/) and vice-versa. 

The framework persists each resource as a [Riak Map](https://docs.basho.com/riak/2.0.1/dev/using/data-types/#Maps). Each resource should have its own converter. The following few sections describe the interface, base converter implementation and what is needed to implement a converter for a specific resource.

## **Interface**

The framework provides an interface called `ResourceConverter` which has to be implemented by all converters. Here is what each method is meant for:

1. `createResource()`: This method creates a new resource object

2. `toResource(@Nonnull RiakMap riakMap)`: This method converts a Riak Map into a resource. A new resource object will be created. This method is used when we are reading data from Riak

3. `toResource(@Nonnull T resource, @Nonnull RiakMap riakMap)`: This method also converts a Riak Map into a resource. But a new resource is not created. Instead, conversion happens using the resource that is passed to this method. This method is used when we are reading data from Riak, and we already have a partially constructed resource object.

4. `toMapUpdate(@Nonnull T resource)`: This method converts a resource into a Riak Map. A new Riak Map will be created. This method is used when we are persisting data into Riak.

5. `toMapUpdate(@Nonnull MapUpdate mapUpdate, @Nonnull T resource)`: This method also converts a resource into a Riak Map. But a new Riak Map is not created. Instead, conversion happens using the Riak Map that is passed to this method. This method is used when we are persisting data into Riak, and we already have a partially constructed Riak Map.

The following are the additional methods required for more complex processing in a converter:

1. `toMapUpdate(@Nonnull @NotEmpty String id)`: There are times when you may need to create a Riak Map with just the resource id. This method is meant for that.

2. `addSetUpdate(@Nonnull @NonnullElements Set<String> values)`: This method creates a new [Riak Set](https://docs.basho.com/riak/2.0.1/dev/using/data-types/#Sets) and adds all the String values of the Set into it. This method is used when we want to add a Riak Set to Riak. All the resources are stored as Riak Maps in the framework. But there may be cases where you want to persist data as a Riak Set. This method is meant for it. 

3. `mergeSetUpdate(@Nonnull @NonnullElements Set<String> oldValues, @Nonnull @NonnullElements Set<String> newValues)`: This method creates a new Riak Set. It also gives instructions to Riak to remove old values and add new values into the Riak Set. This method is used when we want to update existing Riak Sets in Riak

4. `deleteSetUpdate(@Nonnull @NonnullElements Set<String> values)`: This method, as the name suggests, is used to delete a Riak Set from Riak. This method gives instructions to Riak to remove all the values of the Riak Set.

5. `getSet(@Nullable RiakSet riakSet, @Nonnull @NotEmpty String key)`: This method converts a Riak Set into a Java Set of strings. This method is used when the data is persisted as Riak Set in Riak and not as Riak Map.

6. `getResourceIds(@Nullable SearchOperation.Response response)`: All the search methods in the framework hit Riak Solr. Once the search succeeds and a response is returned, we will extract the resource ids. These resource ids will then be used to fire a multi-get-by-ids on Riak, which is the final response of Search operations. This method does the extraction of resource ids from a search response. 

## **Abstract Converter**

Now that we've understood the `ResourceConverter` interface, we can go ahead and create an implementation for all the methods. But wait! Doing this for every resource can become overwhelming and time consuming as the number of resources grow. The `riak-support` project comes to your rescue here and provides a base converter implementation and several handy utility methods. There are 2 main abstract implementation classes: 

* `AbstractResourceConverter`: This is the main base implementation class. It provides complete implementation for most of the methods of `ResourceConverter`.
 
* `AbstractRealmScopedResourceConverter`: This is just an extension of `AbstractResourceConverter` for realm scoped resources.

**Only** the following methods will have to implemented in your converter if you extend one of the above abstract converters:

* `createResource()`
* `toResource(@Nonnull T resource, @Nonnull RiakMap riakMap)`
* `toMapUpdate(@Nonnull MapUpdate mapUpdate, @Nonnull T resource)`

For this, you will have to add the following `riak-support` maven dependency in your project:

        <dependency>
            <groupId>com.covisint.core.support.riak</groupId>
            <artifactId>riak-support</artifactId>
            <version>TRUNK-SNAPSHOT</version>
        </dependency>

----------

## **Converter Helper Methods**

Apart from implementing the interface methods, the `AbstractResourceConverter` also has several helper methods which help in converting your resource. Here are a few examples which convert specific type of attributes in a resource. You can find lots more when you explore on your own:

* `#addInternationalStrings` / `#getInternationalStrings` 
* `#addResourceReference` / `#getResourceReference` 
* `#addMap` / `#getMap`: Methods for Java `Map<String, String>`
* `#addSet` / `#getSet`: Methods for Java `Set<String>`
* `#addString` / `#getString`: Methods for String.There are similar methods for other Java data types

All the `add*` methods convert the resource's attributes into Riak Data Type and then adds this to the Riak Map. All the `get*` methods extract the Riak Data Type from the Riak Map and converts it to the resource's attributes. 

So before you go ahead write code on your own for conversion of different types of attributes, take a look at `AbstractResourceConverter`. You will mostly find what you need. 

----------

## **Your Resource Converter**

As described previously, every resource should have its own converter and you can extend your converter from `AbstractResourceConverter` or `AbstractRealmScopedResourceConverter`.

Here is what a sample resource and its converter looks like:

```java

public class Book extends AbstractRealmScopedResource<Book> {

    private static final long serialVersionUID = 1L;

    @Searchable
    @Sortable
    private String title;

    @Searchable
    private Set<InternationalString> description;

    @Searchable
    private String isbn;

    @Searchable
    private ResourceReference author;

    @Searchable
    private Set<String> tags;

    // Getters and setters

    // #toString, #hashCode and #equals
}

public class BookConverter extends AbstractRealmScopedResourceConverter<Book> {

	// As a good practice these constants should be in a Constants file
	public static final String BOOK_TITLE_FIELD = "title";

	public static final String BOOK_DESCRIPTION_FIELD = "description";

	public static final String BOOK_ISBN_FIELD = "isbn";

	public static final String BOOK_AUTHOR_FIELD = "author";

	public static final String BOOK_TAGS_FIELD = "tags";

	/** {@inheritDoc} */
	@Nonnull
	public Book createResource() {
		return new Book();
	}

	/** {@inheritDoc} */
	@Nonnull
	public Book toResource(@Nonnull Book book, @Nonnull RiakMap riakMap) {
		String stringValue = getString(riakMap, BOOK_TITLE_FIELD);
		if (!Strings.isNullOrEmpty(stringValue)) {
			book.setTitle(stringValue);
		}

		final Set<InternationalString> description = getInternationalStrings(
				riakMap, BOOK_DESCRIPTION_FIELD);
		if (description != null) {
			book.setDescription(description);
		}

		stringValue = getString(riakMap, BOOK_ISBN_FIELD);
		if (!Strings.isNullOrEmpty(stringValue)) {
			book.setIsbn(stringValue);
		}

		final ResourceReference author = getResourceReference(riakMap,
				BOOK_AUTHOR_FIELD);
		if (null != author) {
			book.setAuthor(author);
		}

		final Set<String> tags = getSet(riakMap, BOOK_TAGS_FIELD);
		if (null != tags && !tags.isEmpty()) {
			book.setTags(tags);
		}

		return book;
	}

	/** {@inheritDoc} */
	@Nonnull
	public MapUpdate toMapUpdate(@Nonnull MapUpdate mapUpdate,
			@Nonnull Book book) {
		addString(mapUpdate, BOOK_TITLE_FIELD, book.getTitle());
		addInternationalStrings(mapUpdate, BOOK_DESCRIPTION_FIELD,
				book.getDescription());
		addString(mapUpdate, BOOK_ISBN_FIELD, book.getIsbn());
		addResourceReference(mapUpdate, BOOK_AUTHOR_FIELD, book.getAuthor());
		addSet(mapUpdate, BOOK_TAGS_FIELD, book.getTags());

		return mapUpdate;
	}
}
```

As you see in the code above, all you need to do is implement the 3 methods: `#createResource`, `#toResource` and `#toMapUpdate`. In the `#toResource` and `#toMapUpdate`methods, all you have to do is to use the appropriate utility methods from parent class `AbstractResourceConverter` for every attribute of the resource.

_NOTE: In case you have complex attributes or nested objects, then you will have to deal with the storage and retrieval with custom code. Before this, you will have to decide on the format in which you want to store your data. This will be become especially important when the attribute is `@Searchable`. You have to ensure that the field stored will be searchable in Riak Solr. You can understand more in the Implementation section below_

You can see an example of a resource converter class [here](http://git.covisintrnd.com/eng-iot-core/device-service/blob/master/server/src/main/java/com/covisint/platform/device/server/device/DeviceConverter.java).

<table>
  <tr>
    <td><a href="dao">&lt; 2.4. Data Access Object</a></td>
    <td align="right"><a href="riak-indexer">2.6. Riak Indexer &gt;</a></td>
  </tr>
</table>