<table>
  <tr>
    <td><a href="implementation-core">&lt; 1. Core</a></td>
    <td align="right"><a href="json-readers-writers">1.2. JSON Reader/Writer &gt;</a></td>
  </tr>
</table>
# 1.1. Resources vs Entities

There are two types of POJOs that can live in the core module - resources and entities.  

__Resources__

Resources are just that - resources in the HTTP sense.  They implement ```com.covisint.core.http.service.core.Resource``` and should extend ```com.covisint.core.http.service.core.AbstractResource```.  Most of the time your resources will be realm-scoped and will therefore extend ```com.covisint.core.http.service.core.AbstractRealmScopedResource```.  You will need to make a careful decision here as to whether or not your resource is or is not realm scoped.

__Entities__

Entities are POJOs that are really not resources and cannot extend the resource interface.  An example of an entity is anything that can come over in a request body that instructs the server to take some action.  For example, say we have an authorization API whose job is to authorize an action requested by someone or something.  The input for this API may be a JSON document that contains information about the subject and the requested action.  This JSON document cannot be classified as a resource because it is not persisted, fetched or queries from the server.  It is simply a request for the server to carry out some action.  In this case we have what is an "authorization request entity".  Similarly, the authorization API may send back an authorization response.  Let's assume that this response embeds the original request and appends a ```decision``` property that can have values ```permit``` or ```deny```.  This JSON document also cannot be seen as a resource because it doesn't fit the mold of what a resource really is.  And so what we have is an "authorization response entity".  These entities can be written in Java as simple POJOs that do not extend the resource interface.

The framework supports both resources and plain entities as model classes (although it does favor that your primary goal is to work with resources).

Here is what the authorization request and authorization response entity classes may look like:

```java
public class AuthorizationRequest implements java.io.Serializable {
  
  private static final long serialVersionUID = 1L;

  private String subjectId;

  private String action;

  private java.util.Map<String, Object> context;

  // Getters and setters.

  public int hashCode() {
    // implementation
  }

  public boolean equals(Object obj) {
    // implementation
  }

  public String toString() {
    // implementation
  }

}

public class AuthorizationResponse implements java.io.Serializable {
  
  private static final long serialVersionUID = 1L;

  private AuthorizationRequest request;

  private Result result;

  // Getters and setters.

  public int hashCode() {
    // implementation
  }

  public boolean equals(Object obj) {
    // implementation
  }

  public String toString() {
    // implementation
  }

  public enum Result {
    PERMIT,
    DENY;
  }

}
```

_A note before we continue.  You will notice that these entity classes override ```#hashCode```, ```#equals``` and ```#toString```.  As a matter good practice we should always provide these methods in every domain class we write.  They are used by the framework for certain operations as well as in log statements, caches, useful when maintaining unique collections or checking for the existence of objects in collections, and many more.  Also, javadoc is omitted here for brevity, but it is essential that you fully document every method, field, class and interface that you write._

So we have seen that entity domain classes look like regular POJOs and thats basically all they are.

Now let's take a look at a resource domain class.  For the remainder of this section, we will assume our resource is a book.  Here is what the book resource class may look like:

```java
import com.covisint.core.http.service.core.AbstractRealmScopedResource;
import com.covisint.core.support.constraint.Searchable;
import com.covisint.core.support.constraint.Sortable;
import com.google.common.base.MoreObjects;

...

public class Book extends AbstractRealmScopedResource<Book> {

    private static final long serialVersionUID = 1L;

    @Searchable
    @Sortable
    private String title;

    @Searchable
    private String isbn;

    @Searchable
    @Sortable
    private String author;

    private Format format = Format.PAPERBACK;

    // Getters and setters.

    public String toString() {
        return populateToStringHelper(MoreObjects.toStringHelper(this)).add("title", getTitle())
             .add("isbn",isbn).add("author", author).add("format", format).toString();
    }

    public int hashCode() {
        return getResourceIdentityHashCode();
    }

    public boolean equals(Object obj) {
        return isResourceIdentityEqual(Book.class, obj);
    }

    public enum Format {
        PAPERBACK,
        HARDCOVER,
        KINDLE,
        AUDIO_CD,
        LIBRARY_BINDING;
    }
}
```

First, lets start with the basics: there is only one class required to represent a resource.  This class is used in the body for both requests and responses.  It is sent in the request body when creating (```POST```) or updating (```PUT```).  It is returned in the request body for most HTTP operations.  Therefore there is no sense in creating a BookRequest and BookResponse resource.  So be sure not to do that.

We see that the class extends ```AbstractRealmScopedResource``` which declares that a book is scoped to a realm.  The generic type must be given to the abstract parent class and it is the resource type, ```Book```.  There are some fields that are inherited from the parent class - they are described above in more detail but repeated here for convenience:

* id
* realm
* version
* creation
* creator
* creatorAppId

In addition to these base fields, the book resource class adds some more:

* title
* isbn
* author
* format

These are the basic attributes common to all books.

In the class body there will be a getter method and a setter method for each field declared in the book class.  Then we see, again, the ```#toString```, ```#hashCode``` and ```#equals``` methods since every single domain class should implement these.  Since we extend ```AbstractRealmScopedResource``` which extends ```AbstractResource```, we now get to use some utility methods to help us with the implementation of these three methods:

1. ```#populateToStringHelper```: This method will return a populated ```com.google.common.base.MoreObjects.ToStringHelper``` containing all of the base resource fields.  We then add the book-specific fields to this partially-populated helper class and ultimately call its ```#toString``` method before returning the string value of the book resource.  This helper method saves developers from having to rewrite the same code over and over again.

2. ```#getResourceIdentityHashCode```: This method returns the hashcode for the "resource identity" which is essentially a tuple on the resource id and version.  What this implies is that a resource may be uniquely identified and hashed using only its id and version as input to the hash function.

3. ```#isResourceIdentityEqual```: This method returns true or false if the "resource identity" is equal to the one in the object passed in to the method.  If the object is null, or is not of type ```Book```, then this method will immediately return false.  If the object being passed in is referentially equal to this book (```==```) then this method will immediately return true.  Otherwise, the given book's id and version are compared to this book's id and version and the result is returned.

The abstract resource classes we extend all implement interface ```com.covisint.core.http.service.core.Resource``` which extends ```java.io.Serializable``` so there is no need to explicitly implement the serializable interface.  One note on serializable is that all model classes __MUST__ implement ```java.io.Serializable``` otherwise there will be issues with some of the framework features such as caching the resources in server-side caches.

Resource fields may be annotated with ```@Searchable``` and ```@Sortable``` annotations.  These are way of injecting metadata into the resource class to help the framework automatically discover what fields may or may not be filtered and sorted.  These operations are supported in the framework's search API and so these annotations have no effect on the other CRUD operations.  Also, these annotations only apply to resources that are persisted.  They don't apply to plain entities as the framework does not recognize them as persistent resources.  A deep dive on these annotations, how they work and how the framework treats them will be left to the section on controllers below.  For now just keep in mind that as you design resources that have searchable and sortable properties, annotate them accordingly with these two classes.

Another area of importance when writing your core model classes is interface design.  You are creating classes that will be part of a larger SDK used by both internal and external developers.  The core module of your HTTP service will be published to Maven Central and used by the general population for as long as your service is deployed, running and supported.  Therefore there are some items of code hygiene that you should make sure to address:

1) Source License
The header of each and every source file must contain the [Apache v2 Open Source License](http://www.apache.org/licenses/LICENSE-2.0).  Be sure to append the following section to the header of your Java, XML, JSON, Protobuf, etc files:

```
/*
 * Copyright (C) 2015 Covisint
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
```
 
This license must only appear on source files that will be part of publicly-available libraries.  In our case, these are the core, protobuf and client modules.  The server and webapp modules are only deployed server-side, are never to be seen by external developers, and therefore don't need the Apache v2 license headers. 

2) Javadoc

Every single member of your core classes must be fully javadoc-ed in a clear, verbose manner.  This includes classes, interfaces, enums, all fields, all methods, all method args and return types, all generic types.  The documentation you add will be compiled into javadocs and published internally AND to Maven Central for the world to read and use.  This documentation should be accurate and explain with full clarity what the method does, what it expects as input, how it uses those inputs, the edge conditions and how it deals with bad input, and what it returns and the constraints on its return type.  Later on in this guide we will introduce the constraint annotations that do a lot of auto-documenting for you when places on your method signature elements.  But for now, the key takeaway here is to fully document everything you write.  We have Sonar instruments in place to detect javadoc levels and alert on certain conditions, and the goal is to use these metrics to pass and fail builds as needed.

3) Accessor and Mutator Methods

The guts of all your model classes will be just a bunch of getters and setters.  This is simple enough, but there are considerations and actions you need to take to make even this API fluent and intuitive for outside developers to use:

   * Always return the object instance from within a setter.  This will allow developers to chain calls together when building a resource object from scratch:

```java
  public Book setTitle(String newTitle) {
    title = newTitle;
    return this;
  }

  public Book setIsbn(String newIsbn) {
    isbn = newIsbn;
    return this;
  }
```

From a consumer's point of view, it allows them to chain calls instead of having one call per line:

```java
  Book book = new Book().setCreator("Sam").setTitle("The Old Man and the Sea").setIsbn("1-56619-909-3");

  // vs

  Book book = new Book();
  book.setCreator("Sam");
  book.setTitle("The Old Man and the Sea");
  book.setIsbn("1-56619-909-3");
```

4) Collection Setter Methods

Commonly you will be using collection types for your resource fields.  Maybe a set or a map or an arraylist.  It is always a good idea to give a few different options to the end user for how to set up or append to the collection.  Always ask yourself "how can I make this available with a single method call?" and your answer will come.  Lets say that our book resource contained a collection of reviews for that book.  Assume that a review was represented as a simple string, and that the collection to contain them was a ```java.util.ArrayList```.  Here are some methods to provide developers easy use of your Java API:

```java

import java.util.List;
import com.google.common.collect.Lists;

// javadoc omitted for brevity

...
  
  /* 
   * Unless there is a need not to, always instantiate the list eagerly. It will simplify code
   * that uses it.  It will eliminate many forms of NullPointerExceptions.  Finally it allows 
   * you to declare in your accessor method signature that it will never return a null collection.
   * This in turn will relieve many null-check burdens from consumer developers.
   */
  private List<String> reviews = Lists.newArrayList();

...

    @Nonnull
    @NonnullElements
    @Unmodifiable
    public List<String> getReviews() {
      return reviews;
    }

    @Nonnull
    public Book setReviews(@Nonnull @NonnullElements java.lang.Iterable<String> newReviews) {
        reviews = Lists.newArrayList(newReviews);
        return this;
    }

    @Nonnull
    public Book setReviews(@Nonnull @NonnullElements String... newReviews) {
        reviews = Lists.newArrayList(newReviews);
        return this;
    }

    @Nonnull
    public Book addReview(@Nonnull String review) {
        reviews.add(review);
        return this;
    }

    @Nonnull
    public Book removeReview(@Nonnull @NotEmpty String review) {

        for (final Iterator<String> i = reviews.iterator(); i.hasNext();) {
            if (i.next().getId().equals(review)) {
                i.remove();
            }
        }

        return this;
    }

``` 

The ```@Nonnull```, ```@NonnullElements``` and ```@Unmodifiable``` attributes are shown here just as an example of a good method signature.  They are a declarative way of both documenting and injecting behavior with just a single annotation.  We will cover these constraint annotations later on, but just keep in mind that this is their purpose and they should be used liberally wherever needed.

You can see there are two overloaded setter methods, one which takes an ```Iterable``` and another which takes varargs.  Though they both essentially do the same thing (replace the reviews list with the given items) they each server an important purpose.  The method taking an Iterable can be applied for any collection whatsoever.  If the caller has a Set, or an ArrayList, or a Vector, or LinkedList, etc...there is no additional work needed to convert one type of collection to another.  The method taking varargs can be used when there is only one review to be set into the book, and instantiating a new ArrayList just to add one element to it and then setting that ArrayList into the book would be 3 statements when only one would do.  To put it into code, here is how the Iterable method would look like in practice:

```java

  Book book = // set up book object

  java.util.Set<String> reviews = // the reviews are in a set

  book.setReviews(reviews); // internally, the book's reviews are stored in an ArrayList 
                            // but in this case we made it easier by accepting a Set.

```

Here is how the varargs method would be used in practice:

```java

  Book book = // set up book object

  book.setReviews("I only have 1 review to add!"); // Here we are just placing a single review.

  book.setReviews("Here's a review.", "Here's another!"); // Set two reviews just as easily.
```

Finally, the observant reader will notice that there is actually another quality introduced with the overloaded setter methods.  They apply "defensive programming" by making a copy of the given method argument instead of setting it directly.  Since we don't know who will be using our classes and how, we must take care to apply defensive measures so that we protect the internal state of our objects.  By making a copy of the reviews given to us, we ensure that if the caller's reference to that passed collection were ever used to make modifications to it, we would still be safe with our duplicate copy.  Always think defensively when authoring classes that will be used by a public audience.  There is no way for you to guarantee they will be "nice" to your API.

You can see an example of a resource class that follows all of the above guidelines [here](http://git.covisintrnd.com/eng-iot-core/device-service/blob/master/core/src/main/java/com/covisint/platform/device/core/device/Device.java).

***
__Extension Point: Model Class Hierarchy__

All of the examples given so far have shown only simple, one-level classes.  The AuthorizationRequest and AuthorizationResponse entity classes do not extend anything and are themselves the final class (no subclass).  Book also does not have any subclass.  However, there is nothing stopping a developer from writing a base resource class if needed, abstracting away some common fields that are to be shared by multiple child classes.  The only guideline is that the base class should be declared ```abstract``` and the class name should begin with either ```Base``` or ```Abstract```.  While we are on the topic of inheritance, it has been found to be a good idea __NOT__ to declare your model classes as ```final```.  Although this may initially seem like good practice according to some theories, it is best to simply obey the [open/closed principle](https://en.wikipedia.org/wiki/Open/closed_principle) and leave the class open for extension.
***

<table>
  <tr>
    <td><a href="implementation-core">&lt; 1. Core</a></td>
    <td align="right"><a href="json-readers-writers">1.2. JSON Reader/Writer &gt;</a></td>
  </tr>
</table>