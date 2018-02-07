<table>
  <tr>
    <td><a href="client-factory">&lt; 4.4. Factory</a></td>
    <td align="right"><a href="implementation-nuts-bolts">V Implementation: The Nuts and Bolts &gt;</a></td>
  </tr>
</table>

# 4.5. Client SDK

Till now, we have seen how to create a client and use it. All this is for internal usage, which means, we can use clients like this within Covisint applications since we know that they are already authenticated by some means and they do know the values to pass in the `HttpContext`. But, when you want to provide the clients to the outside world, to integrate the APIs in their applications, then we need to provide them only the public facing APIs. Public clients are interfaces that use the public-facing HTTP services, and therefore their access to those public APIs must be authenticated first.  It is not permissible to allow unauthenticated access to __ANY__ publicly exposed APIs. Public client is like a secure wrapper around the Internal client. You can read more about authentication mechanism in the section [Public clients are interfaces that use the public-facing HTTP services, and therefore their access to those public APIs must be authenticated first.  It is not permissible to allow unauthenticated access to __ANY__ publicly exposed APIs](oauth-client)

## Your Client SDK

When you want to implement SDK for your resource, then you will extend it from the client factory that we talked about in the previous chapter. Below is a sample SDK code:

* Create the `BookSDK` class extending from its factory, `BookClientFactory`:

```java
public class BookSDK extends BookClientFactory {

    /**
     * Constructor.
     * 
     * @param serviceUrl the service url.
     */
    public BookSDK(@Nonnull @NotEmpty String serviceUrl) {
        super(serviceUrl);
    }

    /** {@inheritDoc} */
    public BookClientImpl create() {
        throw new UnsupportedOperationException("Please use #newClient to create new client instances.");
    }

    /**
     * Creates a new book client.
     * 
     * @return returns a new book client.
     */
    @Nonnull
    public BookClient newClient() {
        return new ClientImpl(super.create());
    }

    // Code Part-2 continued below ...
```

* `#BookSDK`: This is the constructor which takes the Public APIs. It just calls the super constructor by passing the URL.
    
* `#create`: We are overriding the #create method from the client factory and making it an unsupported operation. This is because we do not want developers to be able to create a client instance directly from the factory. We are instead recommending the developer to use the `#newClient` method
    
* `#newClient`: This method creates an instance of the client using the `#create` method from the factory. An the client instance is passed to the `ClientImpl` instance. *Note* that this `ClientImpl` is an inner class in the SDK itself. You will see what it is in the next code snippet.

```java

    // ... Code Part-2 continued here

    /** Client interface for the {@link Book} API. */
    public interface BookClient {

        /**
         * Adds a new book.
         * 
         * @param book The book to add.
         * @return Returns a future producing the newly created book. This object contains the server-issued id and
         *         version.
         */
        @Nonnull
        CheckedFuture<Book, ServiceException> add(@Nonnull Book book);

        /**
         * Updates an existing book. The book must already exist, otherwise an error is thrown.
         * 
         * @param book The book to update.
         * @return Returns a future producing the updated book.
         */
        @Nonnull
        CheckedFuture<Book, ServiceException> update(@Nonnull Book book);

        /**
         * Get a resource by ID.
         * 
         * @param id id of the resource to be returned.
         * @return a future containing the requested resource, if it exists.
         */
        @Nonnull
        CheckedFuture<Book, ServiceException> get(@Nonnull @NotEmpty String id);

        /**
         * Delete a book.
         * 
         * @param id id of the resource to be deleted.
         * @return Returns a future producing the deleted book.
         */
        @Nonnull
        CheckedFuture<Book, ServiceException> delete(@Nonnull @NotEmpty String id);

        /**
         * Searches for books based on supported criteria.
         * 
         * @param filter The filter criteria to apply. If <code>null</code> is passed, then no filtering is performed.
         * @param page The pagination criteria to apply. Pass {@link Page#DEFAULT} to return the standard result
         *            collection.
         * @param sortOrder The sort criteria to apply. Leave empty to skip sort operations.
         * @return Returns a future producing the iterable collection of books matching the search criteria.
         */
        @Nonnull
        CheckedFuture<List<Book>, ServiceException> search(@Nullable BookFilterSpec filter, @Nonnull Page page,
                Sort... sortOrder);

        /**
         * Borrow the requested book.
         * 
         * @param bookId The id of the book to borrow.
         * @return Returns a future indicating the completion of this task.
         */
        @Nonnull
        CheckedFuture<Void, ServiceException> borrowBook(@Nonnull @NotEmpty String bookId);

        /**
         * Return the requested book.
         * 
         * @param bookId The id of the book to return.
         * @return Returns a future indicating the completion of this task.
         */
        @Nonnull
        CheckedFuture<Void, ServiceException> returnBook(@Nonnull @NotEmpty String bookId);

        // Code Part-3 continued below ...
```

* Create an inner interface in the SDK called `BookClient`. This interface will be very similar to the Client Interface we saw in the previous chapters. It will have all the method definitions that are there in the Client Interface. But each of these method definitions, will *NOT* have the `HttpContext` argument. Remember we talked in the beginning that the external developers will not have the information about `HttpContext`.

* You will also notice that the `#search` method arguments are slightly different. Instead of `Multimap<String, String>` for search criteria and `SortCriteria` for sort criteria, we will be using `BookFilterSpec` object and `Sort` enum respectively. These are defined more in the next code snippet.


```java
        // ... Code Part-3 continued here

        /** All supported sort fields. */
        enum Sort {

            /** Book title ascending sort. */
            BOOK_TITLE_ASC("+title"),

            /** Book title descending sort. */
            BOOK_TITLE_DESC("-title"),

            /** Book author ascending sort. */
            BOOK_AUTHOR_ASC("+author"),

            /** Book author descending sort. */
            BOOK_AUTHOR_DESC("-author"),

            /** Creation ascending sort. */
            BOOK_CREATION_ASC("+creation"),

            /** Creation descending sort. */
            BOOK_CREATION_DESC("-creation");

            /** Sort field id. */
            private final String id;

            /**
             * Constructor.
             *
             * @param newId The sort field.
             */
            Sort(String newId) {
                id = newId;
            }

            /** {@inheritDoc} */
            public String toString() {
                return id;
            }

        }

        // Code Part-4 continued below ...
```

* We can very well use the `SortCriteria` object to pass the sort criteria to the `#search` method, just like in the Client Interface. But we'll go one step ahead and make it easy for the developer to use a predefined set of sort criteria applicable for your resource. Create an enum for all the `@Sortable` fields of the resource. If you remember the `Book` resource from the [Resources vs Entities](resources-vs-entities) chapter, there were 2 sortable fields - `title` and `author`. The `creation` field automatically becomes sortable because `Book` extends `AbstractRealmScopedResource`. You wil have to create 2 enum constants for each sortable field - one for ascending and the other for descending. The developer can then use this enum and pass to the `#search` method.

```java
        // ... Code Part-4 continued here

        /** Filter criteria for book searches. */
        class BookFilterSpec extends BaseFilterSpec<BookFilterSpec> {

            /** The titles to filter on. */
            private Set<String> titles = new HashSet<>();

            /** The isbn's to filter on. */
            private Set<String> isbns = new HashSet<>();

            /** The authors to filter on. */
            private Set<String> authors = new HashSet<>();

            /**
             * Compares this filter with that one.
             * 
             * @param that The other filter to compare against.
             * @return Returns the comparison result.
             */
            private boolean compareFilters(BookFilterSpec that) {
                return super.equals(that) && Objects.equals(this.titles, that.titles)
                        && Objects.equals(this.isbns, that.isbns) && Objects.equals(this.authors, that.authors);
            }

            /**
             * Gets the titles to filter on.
             * 
             * @return Returns the names to filter on.
             */
            @Nonnull
            @NonnullElements
            @Unmodifiable
            public Set<String> getTitles() {
                return titles;
            }

            /**
             * Sets the titles to filter on.
             * 
             * @param newTitles The titles to filter on.
             * @return Returns this instance.
             */
            @Nonnull
            public BookFilterSpec setTitles(@Nonnull @NonnullElements Set<String> newTitles) {
                titles = new HashSet<>(newTitles);
                return this;
            }

            /**
             * Sets the titles to filter on.
             * 
             * @param newTitles The titles to filter on.
             * @return Returns this instance.
             */
            @Nonnull
            public BookFilterSpec setTitles(@Nonnull @NonnullElements String... newTitles) {
                titles = Sets.newHashSet(newTitles);
                return this;
            }

            /**
             * Adds a new title to filter on.
             * 
             * @param newTitle The new title to add to the filter.
             * @return Returns this instance.
             */
            @Nonnull
            public BookFilterSpec addTitle(@Nonnull @NotEmpty String newTitle) {
                titles.add(newTitle);
                return this;
            }

            /**
             * Gets the isbns to filter on.
             * 
             * @return Returns the isbns to filter on.
             */
            @Nonnull
            @NonnullElements
            @Unmodifiable
            public Set<String> getIsbns() {
                return isbns;
            }

            /**
             * Sets the isbns to filter on.
             * 
             * @param newIsbns The isbns to filter on.
             * @return Returns this instance.
             */
            @Nonnull
            public BookFilterSpec setIsbns(@Nonnull @NonnullElements Set<String> newIsbns) {
                isbns = new HashSet<>(newIsbns);
                return this;
            }

            /**
             * Sets the isbns to filter on.
             * 
             * @param newIsbns The isbns to filter on.
             * @return Returns this instance.
             */
            @Nonnull
            public BookFilterSpec setIsbns(@Nonnull @NonnullElements String... newIsbns) {
                isbns = Sets.newHashSet(newIsbns);
                return this;
            }

            /**
             * Adds a new isbn to filter on.
             * 
             * @param newIsbn The new isbn to add to the filter.
             * @return Returns this instance.
             */
            @Nonnull
            public BookFilterSpec addIsbn(@Nonnull @NotEmpty String newIsbn) {
                isbns.add(newIsbn);
                return this;
            }

            /**
             * Gets the authors to filter on.
             * 
             * @return Returns the authors to filter on.
             */
            @Nonnull
            @NonnullElements
            public Set<String> getAuthors() {
                return authors;
            }

            /**
             * Sets the authors to filter on.
             * 
             * @param newAuthors The authors to filter on.
             * @return Returns this instance.
             */
            @Nonnull
            public BookFilterSpec setAuthors(@Nonnull @NonnullElements Set<String> newAuthors) {
                authors = new HashSet<>(newAuthors);
                return this;
            }

            /**
             * Sets the authors to filter on.
             * 
             * @param newTags The authors to filter on.
             * @return Returns this instance.
             */
            @Nonnull
            public BookFilterSpec setAuthors(@Nonnull @NonnullElements String... newAuthors) {
                authors = Sets.newHashSet(newAuthors);
                return this;
            }

            /**
             * Adds a new author to filter on.
             * 
             * @param newAuthor The new author to add to the filter.
             * @return Returns this instance.
             */
            @Nonnull
            public BookFilterSpec addAuthor(@Nonnull @NotEmpty String newAuthor) {
                authors.add(newAuthor);
                return this;
            }

            /** {@inheritDoc} */
            public boolean equals(Object obj) {
                if (obj == this) {
                    return true;
                }

                if (!(obj instanceof BookFilterSpec)) {
                    return false;
                }

                @SuppressWarnings("unchecked")
                final BookFilterSpec that = (BookFilterSpec) obj;

                return compareFilters(that);
            }

            /** {@inheritDoc} */
            public int hashCode() {
                return Objects.hash(super.hashCode(), titles, isbns, authors);
            }

            /** {@inheritDoc} */
            public String toString() {
                return populateToStringHelper(MoreObjects.toStringHelper(this)).add("titles", titles)
                        .add("isbns", isbns).add("authors", authors).toString();
            }
        }

    } // BookClient interface ends here

    // Code Part-5 continued below ...

```

* Instead of asking the developer to provide the search field name and value via `Multimap<String, String>`, let us give a utility where the developer need not remember what are the different search fields available in a resource. Create a `BookFilterSpec` class holding attributes for each of the `@Searchable` fields. If you remember the `Book` resource, there were 3 searchable fields - `title`, `isbn`, `author`. Since `Book` extends `AbstractRealmScopedResource`, `id` will also be a searchable field.

* You will notice that `BookFilterSpec` extends `BaseFilterSpec`. So `id` attribute will automatically be available with all its get/set methods. Define only the remaining attributes in your filter spec.

* Create all the get* / set* / add* methods. The combinations provided in the above code makes it extremely easy for the developer to set the values.

* You will notice that each of the attributes are Java `List`s. This is because our requirement says each multiple values are allowed in the search fields. If your requirement says just 1 search value allowed, then create the attribute accordingly.

* With this, the `BookClient` interface in the SDK is complete. We will next look at the implementation of this interface.

```java

    // ... Code Part-5 continued here

    /** {@link BookClient} implementation that wraps and delegates to {@link BookClientImpl}. */
    private static final class ClientImpl implements BookClient {

        /** The delegate. */
        private final BookClientImpl delegate;

        /**
         * Constructor.
         *
         * @param newDelegate The delegate to set.
         */
        private ClientImpl(@Nonnull BookClientImpl newDelegate) {
            delegate = newDelegate;
        }

        /**
         * Builds the search criteria SortedSetMultimap based on the given filter spec.
         * 
         * @param filter The filter spec.
         * @return Returns a new SortedSetMultimap of corresponding filter criteria.
         */
        private static SortedSetMultimap<String, String> buildSearchCriteria(BookFilterSpec filter) {

            final SortedSetMultimap<String, String> searchCriteria = TreeMultimap.create();

            if (filter == null) {
                return searchCriteria;
            }

            if (!filter.getIds().isEmpty()) {
                searchCriteria.putAll("id", filter.getIds());
            }

            if (!filter.getTitles().isEmpty()) {
                searchCriteria.putAll("title", filter.getTitles());
            }

            if (!filter.getIsbns().isEmpty()) {
                searchCriteria.putAll("isbn", filter.getIsbns());
            }

            if (!filter.getAuthors().isEmpty()) {
                searchCriteria.putAll("author", filter.getAuthors());
            }

            return searchCriteria;
        }

        /**
         * Build sort criteria based on the given sort order.
         * 
         * @param sortOrder The sort order.
         * @return Returns a new sort criteria instance.
         */
        private static SortCriteria buildSortCriteria(Sort... sortOrder) {

            final SortCriteria.Builder sortBuilder = SortCriteria.builder();

            if (sortOrder != null) {
                for (Sort item : sortOrder) {
                    if (item == null) {
                        continue;
                    }
                    sortBuilder.parseSortField(item.toString());
                }
            }

            return sortBuilder.build();
        }

        /** {@inheritDoc} */
        @Nonnull
        public CheckedFuture<Book, ServiceException> add(@Nonnull Book book) {
            return delegate.add(book, getHttpContext());
        }

        /** {@inheritDoc} */
        @Nonnull
        public CheckedFuture<Book, ServiceException> update(@Nonnull Book book) {
            return delegate.update(book, getHttpContext());
        }

        /** {@inheritDoc} */
        @Nonnull
        public CheckedFuture<Book, ServiceException> delete(@Nonnull @NotEmpty String id) {
            return delegate.delete(id, getHttpContext());
        }

        /** {@inheritDoc} */
        @Nonnull
        public CheckedFuture<Book, ServiceException> get(@Nonnull @NotEmpty String id) {
            return delegate.get(id, getHttpContext());
        }

        /** {@inheritDoc} */
        @Nonnull
        public CheckedFuture<List<Book>, ServiceException> search(@Nullable BookFilterSpec filter, @Nonnull Page page,
                Sort... sortOrder) {

            final SortedSetMultimap<String, String> searchCriteria = buildSearchCriteria(filter);
            final SortCriteria sortCriteria = buildSortCriteria(sortOrder);

            return delegate.search(searchCriteria, sortCriteria, page, getHttpContext());
        }

        /** {@inheritDoc} */
        @Nonnull
        public CheckedFuture<Void, ServiceException> borrowBook(@Nonnull @NotEmpty String bookId) {
            return delegate.borrowBook(bookId, getHttpContext());
        }

        /** {@inheritDoc} */
        @Nonnull
        public CheckedFuture<Void, ServiceException> returnBook(@Nonnull @NotEmpty String bookId) {
            return delegate.returnBook(bookId, getHttpContext());
        }

    }

}
```

* This is the `ClientImpl` we saw previously in the `#newClient` method of the SDK.

* `#ClientImpl`: This is the constructor. In the `#newClient` method, we said, we will pass the Client Instance created from the factory to this `ClientImpl` instance. That is exactly what you see in this constructor. So the `delegate` that you see in the rest of the code is nothing but the instance created from the client factory.

* You will then have to implement all the methods defined in the `BookSDK.BookClient` interface. Not that you will just be calling the corresponding methods from the `BookClientImpl` or `delegate` attribute here. To pass the HttpContext, just use the `#getHttpContext` available in the base client factory.

* Since we used `Sort` enum and `BookFilterSpec` object for `#search` method in the `BookSDK.BookClient` interface, you will now have to convert them internally in your `#search` method to `SortCriteria` and `Multimap<String, String>` respectively. The utility methods for this purpose are `#buildSortCriteria` and `#buildSearchCriteria` respectively.

## How to use the SDK

We have seen a full implementation of the SDK. But nowhere did you see how the authentication is happening. Isn't it? That is because it is handled by the client factory internally. And hence you need to provide a file called `client.conf`. A sample file is shown below.  This file must reside at the root of the consumer application's classpath:

```
clientId = UaQQa8PmQeTvL62nNvWVOtQ
clientSecret = GhCQAfu70Dsy
authServiceBaseUrl = https://apiqa.np.covapp.io/oauth/v3/token
```

The SDK then internally authenticates the API calls and populates all the required `HttpContext` information. Once this is done, use the SDK as specified in the code snippet below:

```java
        // Replace <uri> with the actual public facing URL for the mircroservice
        final String url = "https://<uri>"; 

        // Create an instance of BookSDK.BookClient using the SDK
        final BookSDK.BookClient client = new BookSDK(url).newClient();

        // Let us call the get-by-id method
        CheckedFuture<Book, ServiceException> result = client.get(bookId);
        Book book = result.get();        
```

You can note that the number of lines of code has reduced even further now with the use of SDK comparted to using the client directly. But you will use this Public client i.e., SDK only for calls from external applications

<table>
  <tr>
    <td><a href="client-factory">&lt; 4.4. Factory</a></td>
    <td align="right"><a href="implementation-nuts-bolts">V Implementation: The Nuts and Bolts &gt;</a></td>
  </tr>
</table>