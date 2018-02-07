<table>
  <tr>
    <td><a href="http-service-framework-2-0-0">&lt; Front Page</a></td>
    <td align="right"><a href="principles">II ReSTful Principles &gt;</a></td>
  </tr>
</table>
# I Overview

This document is a guide to help developers understand and use the HTTP service framework developed at Covisint.  It covers a fair amount of the framework's features and explains them in depth.  It is broken up into three main parts:

1. A [Quick Start](quickstart) to allow developers to jump right in and crank out a working HTTP service in just a few minutes.

2. An [Implementation Guide](implementation-basics) which takes the developer through the steps required to build new APIs from scratch.

3. A [Deep Dive](implementation-nuts-bolts) that describes the inner workings of the framework features and shows the developer how to configure and use those features.

Finally, while this framework is "opinionated" in certain regards it does support many ways for the developer to extend the functionality, behavior and configuration when needed.  Therefore, look for sections titled __Extension Point__ that describe these capabilities.

_Note: this guide documents an evolving framework and so applies only to the framework version specified in the title._

## What the framework is

The job of this framework is simple: to speed up development of HTTP services that try to follow the ReST architectural principles.  While the term [microservice](http://martinfowler.com/articles/microservices.html) has grown to be overloaded in recent days, their core design objectives are very valuable and this framework attempts to guide the developer to adopt them.  The main use case served is when there are well defined resources that are persisted to a datastore.

A secondary objective is to promote a homogenous architecture across all HTTP services deployed within the Covisint platform.  From structural design to standard behaviors to uniform constraints, the framework guarantees that all services "look and feel" the same to DevOps engineers, therefore reducing ramp-up time and eliminating any noise that prevents them from the task at hand.

### CRUD + Search

From an API perspective the framework implements, for each resource type, the following HTTP verbs:

| Verb | Operation | URL |
| -------- | -------- | ------- |
| GET | Fetches a resource from the server based on its unique id. | ```http://{host}/{collectionPath}/{resourceId}``` |
| POST | Creates a new resource on the server from the one specified in the request body. | ```http://{host}/{collectionPath}``` |
| PUT | Updates the state of a resource on the server with the one specified in the request body. | ```http://{host}/{collectionPath}/{resourceId}``` |
| DELETE | Deletes a resource from the server based on its unique id. | ```http://{host}/{collectionPath}/{resourceId}``` |

Additionally, a search API is supported:

| Verb | Operation | URL |
| -------- | -------- | ------- |
| GET | Searches for resources on the server based on some filter criteria. | ```http://{host}/{collectionPath}``` |

Search is arguably one of the framework's strongest features.  It supports three modes of operation, which may be mix-and-match as needed:

### Filter criteria
After being told which resource properties are searchable, clients may pass any combination (or none) of those filters in the form of HTTP query parameters.  The framework will see those special parameters, understand that it needs to apply them as filter criteria, and return only matching resources back to the caller.  As an example, take the following resource:

```json
{
  "id": 1,
  "name": "John Smith",
  "email": "jsmith@aol.com"
}
```

Based on this simple resource, if the framework is configured to understand that ```id``` and ```email``` properties are searchable, then the client can issue a GET request on a URL path such as:

```
/persons?email=jsmith@aol.com
```

The framework will return all persons with email address ```jsmith@aol.com```.

### Sort order
Sort order can also be configured on a property by property basis.  Given the same resource above, if ```name``` is configured as a sortable property then the client can issue a GET request on a URL path like:

```
/persons?sortBy=+name or /persons?sortBy=-name
```

which will instruct the framework to sort the results by the person name ascending and descending, respectively.  You can also specify multi-sort chaining, to be able to do things like ```?sortBy=-name&sortBy=+id&sortBy=+email```.

### Pagination
To make working with search results manageable in terms of size, the framework limits how many resources can be returned at a single time.  By default, if not otherwise specified, the first page of 50 results is returned to the client.  The caller can also specify the pagination criteria in terms of ```page``` and ```pageSize``` query parameters.  The former is a 1-based numerical value representing the page index.  ```1``` will request the first page of results, ```10``` requests the 10th page, and so on.  ```pageSize``` indicates how many results to bring back per page.  The maximum value for this parameter is 200, and the default is 50.

A search request with pagination specified may look like this:

```
/persons?page=3&pageSize=100
```

This asks for the third page of 100 results, or in other words, results 201 through 300.

## What the framework is not

There are some areas where the framework is not well suited.  You should make a judgement call when your particular requirements deviate far enough from the framework's core purposes.

1. The persistence store technology is locked down and not easily interchanged.  If you have a requirement that your resources be persisted in a different datastore, this means that _at a minimum_ you will have to replace the entire data access layer with a custom implementation.

2. If you have no need for CRUD operations on resources this is usually telling that the framework may not be of much use to you.

3. If your APIs don't operate on clearly defined resources this is also a tell-tale sign that you will have trouble wedging them into the framework concepts.

4. If your APIs only serve as proxies to other downstream services, HTTP or otherwise, this is also a warning sign that you will be deviating from the core competencies of the framework.

Now having mentioned these points, it is still possible to use the framework to do all of these things; nothing will prohibit you from making it all "work" with enough pushing and shoving.  And actually, using the framework even in such cases will still provide certain advantages over rolling a completely custom HTTP service.  For example, you will still benefit from the marshaling layer to convert your payloads to/from JSON.  You may still benefit from the common logging and metrics support offered by the framework, as well as the span/trace id filter.  All of these things are described at length in sections below.

<table>
  <tr>
    <td><a href="http-service-framework-2-0-0">&lt; Front Page</a></td>
    <td align="right"><a href="principles">II ReSTful Principles &gt;</a></td>
  </tr>
</table>
