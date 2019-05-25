# Protecting ASP.NET Core Microservices with Access Tokens

## Table of Contents

-   <a href="#introduction">Introduction</a>
-   <a href="#architecture">Architecture of the Application</a>
-   <a href="#implementation">Implementation of the Application</a>
    -   <a href="#auth-microservice">The Auth Microservice</a>
    -   <a href="#dummy-microservices">The Protected Microservices</a>
    -   <a href="#cc-client">The Client Credentials Client</a>
    -   <a href="#ro-client">The Resource Owner Client</a>
-   <a href="#demo">Demonstration of the Results</a>
-   <a href="#security">Security Considerations</a>

<a id="introduction"></a>
## Introduction

The goal of this project was to show how microservices implemented with ASP.NET
Core can be protected from unauthorized access through the use of access tokens.

For this purpose, a sample application was created that uses IdentityServer4
[[1]] for access token-based authentication and authorization of two clients
which are trying to access two dummy microservices with different permissions.
The permissions are setup in such a way that only one of the clients is able to
communicate successfully with both of the microservices, while communication
attempts of the second client are accepted by only one of the microservices and
rejected by the other.

<a id="architecture"></a>
## Architecture of the Application

Reading the description of the application in the previous paragraph, a total
of five components can be identified that are needed to implement it:

-   two microservices that should be protected from unauthorized access,
-   two clients that are trying to access them, and
-   one microservice that handles authentication and authorization.

Given these components, a successful communication attempt by one of the clients
with one of the protected microservices generally consists of the following
steps:

1.  The client requests an access token from the auth microservice. To prove its
    identity to the auth microservice, the client sends its credentials with the
    request.
1.  The auth microservice verifies that the credentials which the client sent
    are valid and, finding that they are, returns an access token that is signed
    with the auth microservice's private key to the client.
1.  The client requests the resource it wants to access from the protected
    microservice. To prove that it is authorized to access the microservice, the
    client sends its access token with the request.
1.  To confirm that the access token that the client sent is valid, the
    protected microservice requests the auth mircoservice's public key and uses
    it to check the signature. Finding that the access token is ok and that the 
    permissons that the access token grants (defined in its "scope" attribute)
    are sufficient, the protected microservice then responds to the client's
    request with the requested resource. 

If, at any point during this communication flow, the auth information that the
client sends is found to be invalid or insufficient, the client is notified
about this and denied access to the access token and/or the requested
resource.

More thorough descriptions of this authentication and authorization process can
be found in the *OpenID Connect* [[2]] and *OAuth* [[3]] specifications which
the IdentityServer4 framework implements.

<a id="implementation"></a>
## Implementation of the Application

<a id="auth-microservice"></a>
### The Auth Microservice

The auth microservice is an ASP.NET core application that uses the
IdentityServer4 middleware to process authentication and authorization requests:

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddIdentityServer()
            .AddDeveloperSigningCredential()
            .AddInMemoryApiResources(Config.GetApis())
            .AddInMemoryClients(Config.GetClients())
            .AddTestUsers(Config.GetUsers());
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseDeveloperExceptionPage();
        app.UseIdentityServer();
    }
}
```

The call to `AddDeveloperSigningCredential` tells the middleware to
automatically generate a pair of RSA keys that should be used for signing.

The lines after this call are providing additional information about the
available microservices as well as the clients/users which are are allowed
access each of them. The `Config` class, in which this information is stored,
looks as follows:

```csharp
public static class Config
{
    public static IEnumerable<ApiResource> GetApis() {
        return new List<ApiResource>
        {
            new ApiResource("dummy1", "Dummy Microservice 1"),
            new ApiResource("dummy2", "Dummy Microservice 2"),
        };
    }

    public static IEnumerable<Client> GetClients()
    {
        return new List<Client>
        {
            new Client
            {
                ClientId = "clientCredentialsClient",
                AllowedGrantTypes = GrantTypes.ClientCredentials,
                ClientSecrets = { new Secret("secret".Sha256()) },
                AllowedScopes = { "dummy1", "dummy2" },
            },
            new Client
            {
                ClientId = "resourceOwnerClient",
                AllowedGrantTypes = GrantTypes.ResourceOwnerPassword,
                ClientSecrets = { new Secret("secret".Sha256()) },
                AllowedScopes = { "dummy1" },
            },
        };
    }

    public static List<TestUser> GetUsers()
    {
        return new List<TestUser>
        {
            new TestUser { SubjectId = "1", Username = "alice", Password = "alice", },
            new TestUser { SubjectId = "2", Username = "bob", Password = "bob", }
        };
    }
}
```

While the contents of `GetApis` and `GetUsers` are rather self-explanatory,
there are perhaps one or two questions that might arise when looking at the
`GetClients` methods:

-   **What is the difference between a `Client` and a `User`?**  
    Clients are the applications that are interacting with the microservices.
    In this project, the two clients are two console applications. Users are
    the people that are using the clients.

-   **What are the `AllowedGrantTypes` of the clients?**  
    As the name suggests, the `AllowedGrantTypes` define which types of grants
    a client can request from the auth microservice. *Grants*, in this case, can
    be thought of as being more or less synonymous with *access tokens*
    
    A grant type of `GrantTypes.ClientCredentials` signifies that the client is
    requesting the access tokens for itself, while a grant type of
    `GrantTypesResourceOwnerPassword` signifies that the client is requesting
    the access tokens to represent a specific user (i.e. the *resource owner*).

<a id="dummy-microservices"></a>
### The Protected Microservices

Both of the protected microservices are simple ASP.NET core applications which
contain just a single "Hello world!"-like REST API endpoint:

```csharp
[Route("api/[controller]")]
[Authorize]
public class DummyController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        // Microservice 1:
        return new JsonResult("Hello from DummyMicroservice1!");
        
        //  Microservice 2:
        // return new JsonResult("Hello from DummyMicroservice2!");
    }
}
```

To ensure that authentication and authorization are enforced by the two
applications, the necessary middleware has to be enabled in the startup
code:

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddMvcCore()
            .AddAuthorization()
            .AddJsonFormatters();

        services.AddAuthentication("Bearer")
            .AddJwtBearer("Bearer", options =>
            {
                options.Authority = "http://localhost:5000";
                options.RequireHttpsMetadata = false;
                
                // Microservice 1:
                options.Audience = "dummy1";
                
                // Microservice 2:
                // options.Audience = "dummy2";
            });
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseDeveloperExceptionPage();
        app.UseAuthentication();
        app.UseMvc();
    }
}
```

`options.Authority = "http://localhost:5000"` tells the microservices the
address of the auth microservice from which they should retrieve the public
key that is required to verify the validity of access token signatures.

`options.Audience = "dummyX"` defines the permissions that are necessary to
access the corresponding microservice.

<a id="cc-client"></a>
### The Client Credentials Client

The first of the two clients, which uses a `ClientCredentialsTokenRequest` to
retrieve an access token for itself, is implemented as follows (error
handling code removed to improve readability):

```csharp
public class Program
{
    private static async Task Main()
    {
        // Get access token:
        var tokenClient = new HttpClient();
        var discovery = await client.GetDiscoveryDocumentAsync("http://localhost:5000");
        
        var tokenRequest = new ClientCredentialsTokenRequest
        {
            Address = discovery.TokenEndpoint,
            ClientId = "clientCredentialsClient",
            ClientSecret = "secret",
            Scope = "dummy1 dummy2",
        };

        var tokenResponse = await tokenClient.RequestClientCredentialsTokenAsync(tokenRequest);

        var apiClient = new HttpClient();
        apiClient.SetBearerToken(tokenResponse.AccessToken);

        // Access microservice 1:
        var response1 = await apiClient.GetAsync("http://localhost:5001/api/dummy");
        Console.Write("Response from Microservice1: ");
        string content1 = await response1.Content.ReadAsStringAsync();
        Console.WriteLine(content1);

        // Access microservice 2:
        var response2 = await apiClient.GetAsync("http://localhost:5002/api/dummy");
        Console.Write("Response from Microservice2: ");
        string content2 = await response2.Content.ReadAsStringAsync();
        Console.WriteLine(content2);
    }
}
```

<a id="ro-client"></a>
### The Resource Owner Client

The second of the two clients uses almost the same implementation as the first
one, but instead of a `ClientCredentialsTokenRequest`, it uses a
`PasswordTokenRequest`, which allows it to represent a specific user:

```csharp
public class Program
{
    private static async Task Main()
    {
        // Get user credentials:
        Console.Write("Username: ");
        string username = Console.ReadLine();
        Console.Write("Password: ");
        string password = Console.ReadLine();

        // Get access token:
        var client = new HttpClient();
        var discovery = await client.GetDiscoveryDocumentAsync("http://localhost:5000");
        
        var tokenRequest = new PasswordTokenRequest
        {
            Address = discovery.TokenEndpoint,
            ClientId = "resourceOwnerClient",
            ClientSecret = "secret",

            UserName = username,
            Password = password,
            Scope = "dummy1",
        };

        var tokenResponse = await tokenClient.RequestClientCredentialsTokenAsync(tokenRequest);

        var apiClient = new HttpClient();
        apiClient.SetBearerToken(tokenResponse.AccessToken);

        // Access microservice 1:
        var response1 = await apiClient.GetAsync("http://localhost:5001/api/dummy");
        Console.Write("Response from Microservice1: ");
        string content1 = await response1.Content.ReadAsStringAsync();
        Console.WriteLine(content1);

        // Access microservice 2:
        var response2 = await apiClient.GetAsync("http://localhost:5002/api/dummy");
        Console.Write("Response from Microservice2: ");
        string content2 = await response2.Content.ReadAsStringAsync();
        Console.WriteLine(content2);
    }
}
```

<a id="demo"></a>
## Demonstration of the Results

Once all of the microservices are started, executing the first of the
two clients, i.e. the "Client Credentials Client", produces the following
output:

```
Response from Microservice1: "Hello from DummyMicroservice1!"
Response from Microservice2: "Hello from DummyMicroservice2!"
```

This output shows that the client is able to successfully communicate
with both of the clients, just as the configuration intended.

For the second of the two clients, i.e. the "Resource Owner Client",
communication attempts should only be accepted by the first of the two
microservices and should be rejected by the the second one. Looking at
the client's output, we can see that this also works as intended:

```
Username: bob
Password: bob

Response from Microservice1: "Hello from DummyMicroservice1!"
Response from Microservice2: Unauthorized
```

Using invalid client credentials in any of the two clients immediately causes
the auth microservice to reject all access attempts with the following JSON
response:

```json
{
  "error": "invalid_client"
}
```

Similarly, entering invalid user credentials when prompted for them by the
second client also results in access attempts being rejected:

```json
{
  "error": "invalid_grant",
  "error_description": "invalid_username_or_password"
}
```

If the first client requests an access token that only grants permission to
access the first of the two protected microservice, the client will get past
the auth microservice and the first of the protected microservices just fine,
but will now also be rejected by the second of the projected microservices in
exactly the same way that the second client was:

```
Response from Microservice1: "Hello from DummyMicroservice1!"
Response from Microservice2: Unauthorized
```

Trying to get additional access permissions for `dummy2` with the second client
will fail to make it past the auth microservice again:

```json
{
  "error": "invalid_scope"
}
```

just as sending a `ClientCredentialsTokenRequest` instead of a
`PasswordTokenRequest` from the second client will:

```json
{
  "error": "unauthorized_client"
}
```

<a id="security"></a>
## Security Considerations

While the current implementation is perfectly fine for local development
purposes, there are several things that would have to be changed to bring its
security up to a level that is fit for use in a production environment.

First, and most important of all, use of HTTPS would have to be enabled. Without
it, access tokens that are sent between the different applications can easily be
stolen by malicious actors to gain access to the resources that are supposed to
be protected. 

Similarly, storing user and client credentials as plain-text in the `Config`
class is neither secure nor practical for applications with more users. A much
safer and more realistic approach would be to store them with hashed and salted
passwords in a database.

Many more suggestions for how the security of OpenID Connect and OAuth based
services can be improved are listed in the corresponding sections of their
specifications [[4]][[5]] as well as in *OAuth 2.0 Threat Model and Security
Considerations* RFC [[6]].

[1]: https://github.com/IdentityServer/IdentityServer4
[2]: https://openid.net/specs/openid-connect-core-1_0.html
[3]: https://tools.ietf.org/html/rfc6749
[4]: https://openid.net/specs/openid-connect-discovery-1_0.html#Security
[5]: https://tools.ietf.org/html/rfc6749#section-10
[6]: https://tools.ietf.org/html/rfc6819
