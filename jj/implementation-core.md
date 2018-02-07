<table>
  <tr>
    <td><a href="implementation-basics">&lt; IV Implementation: The Basics</a></td>
    <td align="right"><a href="resources-vs-entities">1.1. Resources and Entities &gt;</a></td>
  </tr>
</table>
# 1. Core

The ```core``` module is where shared classes reside.  This library is a compile-time dependency of both the ```server``` and ```client``` modules.  It is also bundled into the application war during package time.

Because this library is a logical part of the client and therefore may be deployed into a variety of environments (some of which are constrained such as device hardware, gateways, mobile apps and other J2ME deployments) care has been taken to minimize the number of dependencies used by the core module.  You must keep this in mind when writing your core classes; do not be overzealous in adding external libraries.  They may cause unforeseen problems later on.  Keep the core module as simple as possible.

## Package Naming Convention

The convention for naming core resource packages is as follows:

```
com.covisint.platform.{serviceId}.core.{entityOrResourceType}
```

For example:

```
com.covisint.platform.book.core.book
```

This package will contain all book resources or entity classes.  Notice that the service is named "book" as is the resource in this case.  That is perfectly fine.  If we were to add more resources to the book service, say, a publisher resource, we would have a package for that too:

```
com.covisint.platform.book.core.publisher
```

All resource packages should take the singular noun form, i.e. publisher not publishers, book not books.  This is merely a matter of style and is one way to promote a homogenous design.

<table>
  <tr>
    <td><a href="implementation-basics">&lt; IV Implementation: The Basics</a></td>
    <td align="right"><a href="resources-vs-entities">1.1. Resources and Entities &gt;</a></td>
  </tr>
</table>