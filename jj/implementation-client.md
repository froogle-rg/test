<table>
  <tr>
    <td><a href="implementation-webapp">&lt; 3. Webapp</a></td>
    <td align="right"><a href="client-interface">4.1. Interface &gt;</a></td>
  </tr>
</table>
# 4. Client

Any good application will have a client layer. The clients can be in any language that makes it easy for the end user (in this case, the developer) to use the application. The HTTP Service Framework gives a Java client which uses the HTTP protocol, for free. The client masks all the configuration and plumbing from the developer and makes it as simple as calling Java APIs (or methods). Below is a description of the Internal and Public clients 

## Internal:

The internal client is HTTP Client which is usable only within the Covisint network. It is meant for service to service calls, application to service calls etc. The caller should already be authenticated and authorized to use the APIs and expects the user to set the HTTP Context before calling the Java APIs

### [4.1. Interface](client-interface)
### [4.2. Builder](client-builder)
### [4.3. Implementation](client-impl)

## Public:

In an ideal world, we would expose only the public APIs to the end users. And the public client is exactly for this purpose. This will first authenticate the API call and then invoke the respective APIs. Under the wraps, it uses the internal client post authentication and authorization.

### [4.4. Factory](client-factory)
### [4.5. SDK](client-sdk)

<table>
  <tr>
    <td><a href="implementation-webapp">&lt; 3. Webapp</a></td>
    <td align="right"><a href="client-interface">4.1. Interface &gt;</a></td>
  </tr>
</table>