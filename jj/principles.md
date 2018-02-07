<table>
  <tr>
    <td><a href="overview">&lt; I Overview</a></td>
    <td align="right"><a href="quickstart">III Quick Start: Using the Maven Archetype &gt;</a></td>
  </tr>
</table>
# II ReSTful Principles

The framework tries to adhere to the principles of ReST (Representational State Transfer).  The reader is expected to know what these architectural principles are by reading Roy Fielding's [chapter on ReST](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm) in his dissertation.  This is vital, as it will be possible to deviate from these principles within your own implementation if not careful.

An explanation of the benefits gained by adopting ReST in our HTTP services is beyond the scope of this document, but you can find many good articles online on this topic.
 
## Resources

When designing your HTTP service, the first question you must ask is "What are the resources that will be operated on?".  Defining a clear resource is key.  It presents a clear contract in APIs, and modeling the resource properly will aid downstream tasks like defining the various representations that the resource can take.

In the framework, all resources have an abstraction with some common properties:

__id__ 

The identity of the resource should be its primary identifier. This identifier is usually, though not always, persistent.  In the rare case when an identifier may be changed the service that provides access to the resource <strong>MUST</strong> put persistent redirects from the previous identifier to the new one.  This value is always set by the server and therefore is treated as a read-only property when sent in a request entity body.

This property is a string format and typically a [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier).

***
__Extension Point: Custom Resource IDs__

There may be times when the default random UUID is not sufficient for new resources created via POST.  For example, consider the Covisint Unique ID (CUID) service.  Its job is to generate globally unique IDs for some of the legacy IDM components.  In this case, it may be required that the CUID service is used to control the ID assigned to a new resource.  This is possible by extending the framework method which adds new resources, and can replace the auto-generated UUID with a custom one.
***

__version__

All resources have a version that is used, among other things, to hold resource state on the server. These versions should also be used as the basis for a weak entity tag (ETag) for the resource in order to support efficient client-side caching and conditional operations.

This property is a string and is typically a base-64 encoded representation of the server-side state of the resource.  Clients will not manually set this value in resources; typically, this value will be part of resources fetched via ```GET``` and then subsequently sent on resource updates via ```PUT``` operations.

__realm__

The realm in which this resource is located.  Most resources are realm scoped.  Resources are not realm scoped only when they will <strong>ALWAYS</strong> apply to <strong>ALL</strong> tenants within a deployment. If you're unsure if this condition will always hold true, err on the side of caution and make the resource realm scoped.  For security reasons, this value is always set by the server and therefore is treated as a read-only property when sent in a request entity body.

This property is a string, typically 3 - 25 characters in length.

__creation__

The instant, in milliseconds since the epoch, at which this resource was created.  This value is always set by the server and therefore is treated as a read-only property when sent in a request entity body.

This property is a long (number) value.

__creator__

An identifier representing the subject that created this resource.  This could either be an actual person, or it could be an agent acting on behalf of the agent.  If the latter, then this value will be the same as the one in ```creatorAppId```.

This property is a string.

__creatorAppId__

The agent used to create the resource.  For example, this could be a mobile application, another HTTP service, a postman client, or any other HTTP agent.

This property is a string.

## Verbs

Once the resource has been defined correctly, the next step is to determine what HTTP operations (verbs) will be supported on this resource.  Unless you take appropriate measures, the framework will offer ```GET```, ```POST```, ```PUT```, ```DELETE```, and ```search```.  It may be desired, for example, to prevent ```DELETE``` operations on resources.  In this case, the developer will need to take steps to disable that part of the framework to ensure clients can't delete resources.

If there are more operations that need to be supported on the resource besides CRUD, then that is a perfectly normal use case and will require the definition of custom HTTP operations to carry out those operations.  Typically, custom actions on resources should be implemented as ```POST``` methods and are considered "tasks".  More detail on this can be found below.

## Content Negotiation

Once the resource and verbs have been well defined, the next thing to consider is what you want the media types to be and what the representations will look like.  The representation is what is communicated back and forth between client and server.  An explanation on media types is beyond the scope of this document, but it is encouraged to skim over the relevant sections of [RFC-6838](https://tools.ietf.org/html/rfc6838) to gain an understanding of the structure.  At a high level, it is composed of:

```
{primary_type}/{sub_type}.{version}+{structure_type};{attributes}
```

For example:

```
application/vnd.com.covisint.peanut.v1+json;prettyPrint=true
```

Generally, all media types used in Covisint HTTP services should: 

1. Use ```application``` as the primary type.  Unless you really are sending back plain, unstructured text it should be considered that your representation is application-specific.

2. Prefix the subtype with ```vnd``` which indicates a vendor-specific media type.

3. Make sure the subtype contains the FQDN of your service package, up to and including the resource type.

4. Include a version component, which indicates what version of that media type is requested.  This is needed in case your representations change (in a backward compatible way!) and you want to keep existing clients working with the prior version but want to allow them to upgrade to the new version if needed.  The most common use case for this is when new resource properties are made available on a given release of an API.

5. Includes the structure component, in one of the framework's supported structures.  Custom structures may also be supported, but it would be counter-productive in terms of cross-service homogeneity.

Once the representations have been identified and proper media types established, the next step is to model the actual payload.  With JSON structures, this involves creating a detailed JSON schema to govern the JSON entity body.  With protobuf structures, this means creating the IDL source file to define the binary elements of the payload.

***
__Extension Point: Custom Media Type Parameters__

The framework supports custom media type parameters.  These parameters allow clients to request variations of the returned representations as needed.  This may be a way to limit what is returned so that the client only gets what is really necessary.  Or this could be a request to fetch and embed/nest additional data about the resource, above and beyond what is defined as the "default" resource state.  Whatever it is, the server supports this by capturing these media type parameters and making them available to downstream classes so that appropriate action can be taken.
***

<table>
  <tr>
    <td><a href="overview">&lt; I Overview</a></td>
    <td align="right"><a href="quickstart">III Quick Start: Using the Maven Archetype &gt;</a></td>
  </tr>
</table>