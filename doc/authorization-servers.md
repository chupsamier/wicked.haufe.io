# Using Authorization Servers

This page requires some knowledge of OAuth 2.0 and the different OAuth 2.0 flows.

## Types of communcation between systems

When working with APIs, there are two main types of communication:

* Machine-to-machine communication
* End user-to-service communication

When working with machine-to-machine communication, two machines trust each other without an end user or impersonation is needed. For this kind of communication over APIs, wicked offers API Key authentication and the OAuth 2.0 Client Credentials Flow, which do not require any kind of addition authorization: If you have the API Key (or the Client Credentials, i.e. ID and Secret), you are allowed to call the API, usually without any restriction. These flows/authentication mechanisms work "out of the box" with a standard wicked.haufe.io installation; you don't need to do any additional implementation.

For other kinds of communication, where access actually depends on the identity of an end user, other flows should be used, like the OAuth 2.0 Authorization Code Grant, the Implicit Grant Flow or the Resource Owner Password Grant Flow. These flows are three-legged flows additionally involving an Authorization Server which can be used to decide whether a specific end user is allowed to access a resource via the API or not, and/or possibly with which restrictions (e.g. scopes). These flows are fully supported by wicked but make additional use of the default Authorization Server implementation which is shipped with wicked 1.0+.

## Generic Architecture with an Authorization Server

The generic architecture of the wicked API Portal/Gateway including an Authorization Server looks as follows:

![Auth Server Architecture](images/auth-server-architecture.png)

As you can see, the Authorization Server is part of the actual wicked deployment, and the authorization server uses the wicked components to achieve a certain behaviour of the API Gateway. When you registrer an authorization server with an API Portal configuration, this Authorization Server's end points will be published just as any usual API, exposing the Authorization Server as an end point under the API URL.

The main workflow is usually:

1. A client application (which is what you register with the portal to obtain a client ID and possibly secret) realizes it needs access to an API which is secured via wicked
2. Coded into the application is its `client_id` and the URL of the Authorization Server (and of course also the URL of the API which it wants to access)
3. The application accesses the authorization server via the end point which is exposed via Kong; this can either be an API call itself (using e.g. the Resource Owner Password Grant) or a user agent redirect (using the Implicit Grant)
4. The Authorization Server makes sure it knows the client which is identified by the client ID
5. In most cases, the Authorization Server will delegate the actual identification of the end user to a third party IdP, e.g. ADFS, a SAML IdP or some other identity provider the Authorization Server knows how to talk to. The simples case is that the Authorization Server checks the wicked user database for a username and password combination (any `local` auth method).
6. As soon as the Auth Server knows the identity of the end user and has verified that this particular user is allowed to access the API ("Authorize"), the Authorization Server will use Kong to author an access token
7. Kong does a couple of additional checks (e.g. "does the application have a valid subscription to the API?") and then issues the required type of access token

This flow is the same for both the Implicit Grant Flow and the Resource Owner Password Grant Flow; the only thing which differs is the way the access tokens are passed on, and whether you get a refresh token or not.

## The Authorziation Code Grant

The Authorization Code Grant is the "classical" OAuth2.0 flow. It is the recommended flow for all types of clients: Confidential clients such as classical web applications, Single Page applications (also known as browser based applications) and also for Native or Mobile applications. Depending on the client type, the Authorization Code Grant behaves slightly different in some details, which are described later on in this section.

**Example**: Configuration:

1. A web application `some-web-app` has registered in the API Portal and given its redirect URI as `https://some.web.app/path`.
2. For this web application, a subscription to the API `backend-api` (which supports the OAuth 2.0 Implicit Flow) is created
 
Runtime:

1. The web application is called by an end user
2. There is no access or refresh token token currently stored in the storage (session, HTML5,... depending on the client type)
3. The app sends the end user to the authorization server
4. The authorization server does what it needs to authenticate the user (depending on the configured identity providers)
5. The auth server authorizes the end user (depending on the identity provider and in parts on your business logic)
6. The auth server tells the Kong Adapter to create an access token
7. The auth server redirects the end user back to the application, using the registered redirect URI of the application: `https://some.web.app/path?code=<authorization code>`
8. The application uses its client ID (and mostly client secret), combined with the authorization code, and asks the Token endpoint for an access token
9. If successful, the authorization server's token endpoint will return an access token, and mostly also a refresh token

The web application can now use the access token to safely access the backend API using an `Authorization: Bearer <access token>` header in its AAPI calls. The identity of the end user (authenticated user id) and the scopes (authenticated scopes) are not changeable by the application, but are injected by Kong when proxying the call to the backend API, which is just what we want.

The access token is short lived. When the access token is invalid, the application can either make use of the refres token (if available) to get continued access to the API, or, it can use the "silent refresh" method to get a new authorization code which can in turn be exchanged for a new access token.

The Authorization Code Grant can be used with any configured [Auth Method](auth-methods.md).

### Specialties by Client Type

To get the most secure architecture and OAuth2 flows possible, wicked has to know the nature of each API client. Wicked distinguishes the following three types:

* Confidential Clients
* Public Clients - Single Page Applications (Browser Based Applications)
* Public Clients - Native/Mobile Applications

The following sections describe the specialties which apply on each of these client types when using the Authorization Code Grant.

#### Confidential Clients

A confidential client is a client which can keep secrets in its backend; examples for such applications are classical typical session based web applications with a dedicated backend.

There are no special requirements for such applications to use the Authorization Code Grant. It's still **recommended** that the [PKCE extension](oauth-pkce.md) is used.

#### Public Clients (Single Page Applications/Browser Based Applications)

Single Page Applications/Browser Based Applications are considered public clients as the cannot store any secrets in its code. The code is typically fully transferred to the user's user agent (browser), and thus cannot contain other secret information. Such applications can still use the Authorization Code Grant, but with the following restrictions:

* [The use of the PKCE extension is mandatory](oauth-pkce.md)
* The application will not receive a refresh token; to refresh the access token, the [silent refresh method](oauth-silent-refresh.md) must be applied

#### Public Clients (Native/Mobile Clients)

Another type of public clients are native or mobile applications, such as Windows/macOS applications, or mobile iOS or Android Apps. These are inherently more secure than Single Page Applications, but must still be considered public, as it's technically feasible to reverse engineer the applications and extract secrets from them.

Still, such applications are usually reasonably able to keep runtime data secure. When using the Auhthorization Code Grant, the following applies:

* [The use of the PKCE extension is mandatory](oauth-pkce.md)
* The application will receive a refresh token which can be use to perform a token refresh completely using the backend channel; the application does not need a renewed authentication/authorization step as long as it has access to a refresh token

## The Implicit Grant Flow

**Important**: The Implicit grant is considered deprecated by the IETF. The replacement flow for browser based applications is now the Authorization Code Grant, including the [PKCE extension (Proof Key for Code Exchange)](oauth-pkce.md).

The Implicit Grant Flow is particularly useful for Single Page Applications (SPAs) which need secure access to a backend API. Remember: SPAs are usually pure client side JavaScript and as such are not able to keep any kind of secret, and are vulnerable for manipulations (aka public applications). Using API Keys and/or the Client Credentials Flow is out of the question, as these need a "secret" (either the API key or the Client Secret).

The Implict Grant Flow in contrast only needs you to store the client ID with the application; the application in turn is registered with the API portal, where it has specified a "redirect URI". When wanting an end user to authorize, the web application redirects to the authorization server (stating its client ID in the redirect, e.g. `https://api.yourcompany.com/auth-server/api/whatever?client_id=348762378463487563875683945`). The Authorization server then authenticates the user (with whatever means you need), checks that the authenticated user is allowed to access the API (authorization) and then redirects back to the web application, passing an access token in the fragment of the redirect URI.

**Example**: Configuration:

1. A web application `some-web-app` has registered in the API Portal and given its redirect URI as `https://some.web.app/path`.
2. For this web application, a subscription to the API `backend-api` (which supports the OAuth 2.0 Implicit Flow) is created
 
Runtime:

1. The web application is called by an end user
2. There is no access token currently stored in the (HTML5) local storage of the web app
3. The web app sends the end user to the authorization server
4. The authorization server does what it needs to authenticate the user
5. The auth server authorizes the end user (if he is authorized; this is your business logic)
6. The auth server has the Kong Adapter create an access token
7. The auth server redirects the end user back to the application, using the registered redirect URI of the application: `https://some.web.app/path#access_token=3498348374983794934&expires_in=3600&token_type=bearer``

The web application can now use the access token to safely access the backend API using an `Authorization: Bearer <access token>` header in its AJAX calls. The identity of the end user (authenticated user id) and the scopes (authenticated scopes) are not changeable by the SPA, but are injected by Kong when proxying the call to the backend API, which is just what we want.

The access token is short lived, and cannot be refreshed. When the access token is invalid, the SPA has to re-authenticate and re-authorize again using the Authorization Server.

The Implicit Grant can be used with any configured [Auth Method](auth-methods.md).

## The Resources Owner Password Grant Flow

As you may have realized, the Implicit Grant Flow requires a user agent (browser type) with its application which can follow redirects and keep a session. In cases where this is not the case using the Resource Owner Password Credentials Flow may be advisable.

The Flow name is a little misleading depending on the use case, because in some cases it's just used as a standard means to establish identity, to in turn enable the Authorization Server to authorize the server.

The main difference to the Implicit Flow is that everything can be done using APIs, and thus also works with native applications. The drawback is that the native application needs to collect username and password (which is done solely on the Authorization Server/IdP with the implicit grant flow) and send those to the Authorization Server.

The Authorization Server takes the user name and password and checks those against an IdP (can be anything), and if and only if these credentials check out, the client ID is taken with the user's authenticated identity to create an access token for this user and a specific scope (if applicable).

In addition to an access token, a refresh token is also passed on.

The Resource Owner Password Grant can only be used with [local auth methods](auth-local.md) and in some cases, also with [the external auth method](auth-external.md).

**Important:** The Resource Owner Password Grant is not a recommended flow for any typical use case of OAuth 2.0 anymore. It used to be recommended for mobile applications, but it has been superceded by the Authorization Code Grant with the PKCE extension (see above). It can still be of use for certain integration scenarios where an end user identity is needed, but there is no way to provide an interactive login.

### Specialties per Client Type

* **Confidential Clients** must present both their `client_id` and `client_secret` when requesting a token, or when refreshing a token
* **Public Clients** must only present the `client_id` when requesting or refreshing a token

## Further reading

To know more about the wicked implementation of an authorization server, read more

* [Auth Methods](auth-methods.md)
* [Registration Pools](registration-pools.md)
