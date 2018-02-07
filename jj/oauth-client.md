<table>
  <tr>
    <td><a href="dropwizard-metrics">&lt; 9. Server Metrics with Dropwizard</a></td>
    <td align="right"><a href="jetty-development">11. Embedded Development Server with Jetty &gt;</a></td>
  </tr>
</table>
# 10. OAuth Enabled SDKs with Google OAuth2 Client

As you discovered in the [Client](implementation-client) chapter, the Java clients you build will have two flavors - [internal](http://git.covisintrnd.com/eng-core/http-service-framework/wikis/http-service-framework-2-0-0#internal) and [public](http://git.covisintrnd.com/eng-core/http-service-framework/wikis/http-service-framework-2-0-0#external).  Public clients are interfaces that use the public-facing HTTP services, and therefore their access to those public APIs must be authenticated first.  It is not permissible to allow unauthenticated access to __ANY__ publicly exposed APIs.  The framework employs the [OAuth2](https://tools.ietf.org/html/rfc6749) authentication standard and internally uses the [Google OAuth2 Java Client](https://developers.google.com/api-client-library/java/google-api-java-client/oauth2) to communicate with the authorization and token services and negotiate the authentication.  The framework takes care of all of this behind the scenes, without any coding effort on the part of the developer.  Here is the general sequence describing the steps taken to do this:

1. The consumer application instantiates the client SDK object.

2. During the instantiation process, the framework checks to see if authorization is enabled for that particular client.  See the ```com.covisint.core.http.service.client.BaseResourceClientFactory```'s ```#setAuthorizationEnabled``` and ```#getAuthorizationEnabled``` methods.

3. If authorization is enabled, the framework will build a new ```com.covisint.core.http.service.client.oauth.OAuthTokenSupplier``` and pass it the auth configuration details.  These are by default retrieved from a property file on the classpath named ```client.conf``` (see below for typical content) but it is possible to override these (see extension point below).

4. Once the token supplier is created, the framework will request for an initial access token from the OAuth server.  It will hold onto this access token as it will be used for any subsequent API calls to the server.

5. The access token response contains a value indicating the lifetime of that token.  Requests issued beyond that lifetime will be rejected as the token will have expired.  To address token expiry, the framework will periodically send token refresh requests ahead of the expiry time.  This will retrieve a fresh token from the oauth server and ensure uninterrupted access to the APIs used by the client.

At this point the framework has fully initialized the client SDK and injected a special Apache ```org.apache.http.HttpRequestInterceptor``` that handles all aspects of supplying the authorization token in the ```Authorization``` header.

A sample ```client.conf``` file is shown below.  This file must reside at the root of the consumer application's classpath:

```
clientId = UaQQa8PmQeTvL62nNvWVOtQ
clientSecret = GhCQAfu70Dsy
authServiceBaseUrl = https://apiqa.np.covapp.io/oauth/v3/token
```
  
As shown above, this file must contain the client id and secret pair that was issued to the consumer application.  It must also specify the oauth service base URL.

***
__Extension Point: Custom OAuth Credential Providers__

The default method to supply authorization configuration is via a ```client.conf``` file.  It is possible to override this and provide a custom way for your application to provide these details.  To do so, your resource client factory implementation will simply need to override the method ```#getAuthConfigProvider``` and provide a custom ```com.covisint.core.http.service.client.oauth.AuthConfigurationProvider```.  This provider interface requires some basic methods to be implemented:

1. ```#getClientId```: Returns the consumer's client id
2. ```#getClientSecret```: Returns the consumer's secret (aka password)
3. ```#getAuthServiceBaseUrl```: Returns the base URL to the oauth service
4. ```#getApplicationId```: Optional.  Returns the id assigned to the consumer application.

***



<table>
  <tr>
    <td><a href="dropwizard-metrics">&lt; 9. Server Metrics with Dropwizard</a></td>
    <td align="right"><a href="jetty-development">11. Embedded Development Server with Jetty &gt;</a></td>
  </tr>
</table>