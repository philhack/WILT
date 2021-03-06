# Resources

* Simple OAuth getting started: https://identityserver.github.io/Documentation/docs/overview/simplestOAuth.html
* Annoucing IdentityServer 4: http://leastprivilege.com/2016/01/11/announcing-identityserver-for-asp-net-5-and-net-core/
* Client samples: https://github.com/IdentityServer/IdentityServer3.Samples/tree/master/source/Clients
* JWT: http://bitoftech.net/2014/10/27/json-web-token-asp-net-web-api-2-jwt-owin-authorization-server/
* JWT Authentication in ASP.NET Web API: http://bitoftech.net/2015/02/16/implement-oauth-json-web-tokens-authentication-in-asp-net-web-api-and-identity-2/
* JWT debugger: https://jwt.io/


# OpenId Authentication

## interactions

**website**

* Client: GET client's /Home/UserLogin 
* Server: GET /identity/.well-known/openid-configuration, returning the details of the server
* Server: GET /identity/.well-known/jwks
* Client: Redirect to /identity/connect/authorize
* Server: GET /identity/connect/authorize?client_id=MvcPrototype&redirect_uri=https%3a%2f%2fmvcprototype%2f&response_mode=form_post&response_type=id_token+token&scope=openid+profile+role+publicWebsite+miApi&state=OpenIdConnect.AuthenticationProperties%3dnkKmPYwrkOBWDojiMCDNqI6_Xo1Ms12iPebp7lvOku9I4sp2YNYVfZL4SokzhB8kS4m_Lu7WvjXTFCkKoVF9PJJeYyNii-cURC4vyXXTG7tpLc06eFeszB1kxM65DTHSmTCZiMbvRH6b29-pnvVZkWsvKdwIUgdmHJAal_o5AHk5JUG4wQlcyJZZFALZAhapMzuQApxTbW-f7ohYrK1owg&nonce=635882920865285418.MmMxMzllNzUtNTRkOS00MjBiLWE4OWMtMzEzZWQ2YzZjNTgyYTNiNzVkODItYmMxZS00Y2U1LWFhYmQtZjg1MDgyYmExNDVk
* Server: POST /identity/connect/consent?client_id=MvcPrototype&redirect_uri=https%3A%2F%2Fmvcprototype%2F&response_mode=form_post&response_type=id_token%20token&scope=openid%20profile%20role%20publicWebsite%20miApi&state=OpenIdConnect.AuthenticationProperties%3DnkKmPYwrkOBWDojiMCDNqI6_Xo1Ms12iPebp7lvOku9I4sp2YNYVfZL4SokzhB8kS4m_Lu7WvjXTFCkKoVF9PJJeYyNii-cURC4vyXXTG7tpLc06eFeszB1kxM65DTHSmTCZiMbvRH6b29-pnvVZkWsvKdwIUgdmHJAal_o5AHk5JUG4wQlcyJZZFALZAhapMzuQApxTbW-f7ohYrK1owg&nonce=635882920865285418.MmMxMzllNzUtNTRkOS00MjBiLWE4OWMtMzEzZWQ2YzZjNTgyYTNiNzVkODItYmMxZS00Y2U1LWFhYmQtZjg1MDgyYmExNDVk
* Client: POST /, Validate token
* Server: GET /identity/connect/userinfo, returns user info, {"sub":"iamuser@marketinvoice.com","name":"A User","id":"1","role":"Seller"}
* Client: Redirect to /Home/UserLogin
* Client: GET /Home/UserLogin

**mobile app**

* Authorisation endpoint: https://<authserver>/identity/connect/authorize?client_id=<clientid>&scope=openid%20profile%20role%20mobileApi&response_type=code%20id_token&nonce=xxxd1&redirect_uri=https://marketinvoice.com
* user login page
* user submit id and pwd
* user gets redirected to redirect_uri with code, id_token, and scopes

```
https://www.marketinvoice.com/#code=xxxxx&id_token=<token>&scope=openid%20profile%20role%20mobileApi&session_state=<session state>
```

**Getting refresh token in hybrid flow**

Hybrid flow allows to request a combination of identity token, access token, and code via the front channel using either a fragment encoded redirect (native and JS based clients) or a form post (server-based web applications). The client can make immediate use of an identity token to get access to the user's identity but also retrieve an authorisation code that can be used (by a back end service) to request a refresh token and thus gaining long lived access to resources

The client needs to have client_id, client_secret, and code to call the token endpoint to get a refresh_token


## Setting user claims

```csharp
public override Task GetProfileDataAsync(ProfileDataRequestContext context)
{
    // issue the claims for the user
    using (var db = new MiDataContext())
    {
        var subject = context.Subject.GetSubjectId();
        var user = db.Users.SingleOrDefault(u => u.Email == subject);
        if (user != null)
        {
            var claims = new List<Claim>
            {
                new Claim(Constants.ClaimTypes.Subject, user.Email),
                new Claim(Constants.ClaimTypes.Name, user.Name),
                new Claim(Constants.ClaimTypes.Id, user.UserId.ToString()),
                new Claim(Constants.ClaimTypes.Role, user.Role.Name)
            };
            context.IssuedClaims = claims;
        }

    }

    return Task.FromResult(0);
}

```

# Identity Server client

## required libraries

### for clients

#### Client Credentials flow

```powershell
install-package IdentityModel
install-package IdentityServer3
install-package IdentityServer3.AccessTokenValidation
```

#### Authorisation Code / Implicit flow
```powershell
install-package IdentityModel
install-package IdentityServer3
install-package IdentityServer3.AccessTokenValidation
install-package Microsoft.Owin.Security.OpenIdConnect
```

#### To protect APIs
```powershell
install-package IdentityModel
install-package IdentityServer3.AccessTokenValidation
```



## Set Up for OWIN

### Authorzation / Implicit flow

```csharp
const string clientId = "client id";
const string accessTokenEndpoinot = "token endpoint";
const string clientSecret = "client secret";
const string userInfoEndpoint = "user info endpoint";
const string identityServer = "https://identityserverlocal.marketinvoice.ninja/identity";
const string redirectUri = "https://mvcprototype/";
const string requestingScopes = "openid profile role publicWebsite offline_access";
const string requestingResponseTypes = "code id_token token";

AntiForgeryConfig.UniqueClaimTypeIdentifier = IdentityServer3.Core.Constants.ClaimTypes.Subject;
app.UseCookieAuthentication(new CookieAuthenticationOptions { AuthenticationType = "Cookies" });
app.UseOpenIdConnectAuthentication(new OpenIdConnectAuthenticationOptions {
    Authority = identityServer,
    ClientId = clientId,
    Scope = requestingScopes,
    ResponseType = requestingResponseTypes,
    RedirectUri = redirectUri,
    SignInAsAuthenticationType = "Cookies",
    UseTokenLifetime = true,

    Notifications = new OpenIdConnectAuthenticationNotifications {

        AuthorizationCodeReceived = async n =>
        {
            var tokenClient = new TokenClient(accessTokenEndpoinot, clientId, clientSecret);
            var tokenResponse = await tokenClient.RequestAuthorizationCodeAsync(n.Code, n.RedirectUri);

            var userInfoClient = new UserInfoClient(new Uri(userInfoEndpoint), tokenResponse.AccessToken);
            var userInfoResponse = await userInfoClient.GetAsync();

            // create new identity
            var id = new ClaimsIdentity(n.AuthenticationTicket.Identity.AuthenticationType);
            id.AddClaims(userInfoResponse.GetClaimsIdentity().Claims);
            id.AddClaim(new Claim("id_token", n.ProtocolMessage.IdToken));
            id.AddClaim(new Claim("access_token", tokenResponse.AccessToken));
            id.AddClaim(new Claim("expires_at", DateTime.Now.AddSeconds(tokenResponse.ExpiresIn).ToLocalTime().ToString()));
            id.AddClaim(new Claim("refresh_token", tokenResponse.RefreshToken));
            id.AddClaim(new Claim("sid", n.AuthenticationTicket.Identity.FindFirst("sid").Value));

            n.AuthenticationTicket = new AuthenticationTicket(new ClaimsIdentity(id.Claims, n.AuthenticationTicket.Identity.AuthenticationType), n.AuthenticationTicket.Properties);
        },

        RedirectToIdentityProvider = p =>
        {
            if (p.ProtocolMessage.RequestType == OpenIdConnectRequestType.LogoutRequest)
            {
                var idTokenHint = p.OwinContext.Authentication.User.FindFirst("id_token");

                if (idTokenHint != null)
                {
                    p.ProtocolMessage.IdTokenHint = idTokenHint.Value;
                }
            }

            return Task.FromResult(0);
        },
    }
});

```

### To protect APIs

```csharp
const string identityServer = "identity server uri";
const string requiredScope = "required scope";
app.UseIdentityServerBearerTokenAuthentication(new IdentityServerBearerTokenAuthenticationOptions() {
    Authority = identityServer,
    RequiredScopes = new[] { requiredScope }
});
```

Then in start up

```csharp
IdentityServerConfigure.Configure(app);
WebApiConfig.Register(config);
```


## Flows

```csharp
public enum Flows
{
    AuthorizationCode = 0,
    Implicit = 1,
    Hybrid = 2,
    ClientCredentials = 3,
    ResourceOwner = 4, //resource owner password credential flow
    Custom = 5,
}
```

## Refresh token

* Supported for the following flows: authorization code, hybrid and resource owner password credential flow.
* code flow will call the token endpoint with client_id, client_secret, authorization code, and redirectUricalls. The token response will include id_token, access_token, and refresh_token

```csharp
AuthorizationCodeReceived = async n =>
{
    // use the code to get the access and refresh token
    var tokenClient = new TokenClient("token endpoint", "client id", "client secret");
    var tokenResponse = await tokenClient.RequestAuthorizationCodeAsync(n.Code, n.RedirectUri);

    // use the access token to retrieve claims from userinfo
    var userInfoClient = new UserInfoClient(new Uri("user info endpoint"), tokenResponse.AccessToken);
    var userInfoResponse = await userInfoClient.GetAsync();

    // create new identity
    var id = new ClaimsIdentity(n.AuthenticationTicket.Identity.AuthenticationType);
    id.AddClaims(userInfoResponse.GetClaimsIdentity().Claims);

    id.AddClaim(new Claim("id_token", n.ProtocolMessage.IdToken));
    id.AddClaim(new Claim("access_token", tokenResponse.AccessToken));
    id.AddClaim(new Claim("expires_at", DateTime.Now.AddSeconds(tokenResponse.ExpiresIn).ToLocalTime().ToString()));
    id.AddClaim(new Claim("refresh_token", tokenResponse.RefreshToken));
    id.AddClaim(new Claim("sid", n.AuthenticationTicket.Identity.FindFirst("sid").Value));

    n.AuthenticationTicket = new AuthenticationTicket(new ClaimsIdentity(id.Claims, 
        n.AuthenticationTicket.Identity.AuthenticationType), n.AuthenticationTicket.Properties);
},
```


### Response types

```csharp
// authorization code flow
public const string Code = "code";

// implicit flow
public const string Token        = "token";
public const string IdToken      = "id_token";
public const string IdTokenToken = "id_token token";

// hybrid flow
public const string CodeIdToken      = "code id_token";
public const string CodeToken        = "code token";
public const string CodeIdTokenToken = "code id_token token";
```

# Troubleshootings

## Endless redirection

The protected page keeps redirecting the browser to the login server

Add cookie authentication to OWIN

```csharp
app.UseCookieAuthentication(new CookieAuthenticationOptions {
    AuthenticationType = "Cookies"
});
```

## AntiForgeryToken() error

### nameidentifier doesn't exist

"A claim of type 'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier' or 'http://schemas.microsoft.com/accesscontrolservice/2010/07/claims/identityprovider' was not present on the provided ClaimsIdentity"

Specify ClaimTypes.Subject as unique claim type identifier

```csharp
AntiForgeryConfig.UniqueClaimTypeIdentifier = IdentityServer3.Core.Constants.ClaimTypes.Subject;
```

### "A claim of type 'sub' was not present on the provided ClaimsIdentity."

Make sure you have added subject claims on the server

```csharp
var claims = new List<Claim>
{
    new Claim(Constants.ClaimTypes.Subject, user.Email),
    new Claim(Constants.ClaimTypes.Name, user.Name),
    new Claim(Constants.ClaimTypes.Id, user.UserId.ToString()),
    new Claim(Constants.ClaimTypes.Role, user.Role.Name)
};
context.IssuedClaims = claims;
```

### ERROR Invalid flow for client: Implicit

ResponseTypeToFlowMapping
```csharp
{
    { ResponseTypes.Code, Flows.AuthorizationCode },
    { ResponseTypes.Token, Flows.Implicit },
    { ResponseTypes.IdToken, Flows.Implicit },
    { ResponseTypes.IdTokenToken, Flows.Implicit },
    { ResponseTypes.CodeIdToken, Flows.Hybrid },
    { ResponseTypes.CodeToken, Flows.Hybrid },
    { ResponseTypes.CodeIdTokenToken, Flows.Hybrid }
};
```



    // Overview
    You need to have 1) Clients, 2) Scopes, and 3) Claims

    Client

    It has client id and name, specifying flow, allowed scopes, client secret, and access token lifetime

    ex) admin, Admin, "openid, profile, role, api", hasehd_secret, token_lifetime

    Scope

    You have standard scopes like openid, profile and role, and custom scopes. A client's allowed scopes should be present in the table

    ex) openid, standard

    Claim

    A claim that would be included in the token. For example, if you want to include role claim, Claims table should have role entry.

    ex) role


    // adding role claims 
    Scope should have Scope claim of "role"

    For example, if you want to add roles for api scope, api scope needs to have role scope claim.

    var scopes = con.Query<Scope>("select * from Scopes where Enabled = 1 and Name in @names", new { names = scopeNames });
    foreach (var sScope in scopes)
    {
        var scope = new Scope
        {
            Enabled = localScope.Enabled,
            Name = localScope.Name,
            DisplayName = localScope.DisplayName,
            Description = localScope.Description,
            Type = localScope.ScopeType
        };

        var claims = con.Query<Claim>("select * from Claims where ScopeId = @scopeId", new {scopeId = scope.Id}).ToList();
        if (localClaims.Any())
        {
            scope.Claims = localClaims.Select(c => new ScopeClaim(c.Name, true)).ToList();
        }

        scopes.Add(scope);
    }


**Resources**

* https://developer.linkedin.com/docs/oauth2
* http://www.oauthforaspnet.com/providers/linkedin/
* https://developer.linkedin.com/docs/signin-with-linkedin
* image resource: https://developer.linkedin.com/downloads

**Setting up**

1. Create an application. https://www.linkedin.com/secure/developer?newapp=

   Ensure the "OAuth 2.0 Redirect URLs" field for your application contains a valid callback URL to your server that is listening to complete your portion of the authentication workflow.
   https://pstage.marketinvoice.ninja/auth/linkedin

2. Request an Authorization Code
   
   Redirect the user to LInkedIn's OAuth 2.0 authorisation endpoint. 
   https://www.linkedin.com/oauth/v2/authorization

   example
   
   https://www.linkedin.com/oauth/v2/authorization?response_type=code&client_id=123456789&redirect_uri=https%3A%2F%2Fwww.example.com%2Fauth%2Flinkedin&state=987654321&scope=r_basicprofile

3. Exchange Authorization Code for an Access Token

   POST "x-www-form-urlencoded" 
   
   https://www.linkedin.com/oauth/v2/accessToken
   
   ```
   POST /oauth/v2/accessToken HTTP/1.1
   Host: www.linkedin.com
   Content-Type: application/x-www-form-urlencoded
  
   grant_type=authorization_code&code=987654321&redirect_uri=https%3A%2F%2Fwww.myapp.com%2Fauth%2Flinkedin&client_id=123456789&client_secret=shhdonottell   
   ```
   
4. Make authenticated request
5. Retrieve basic profile data

   https://api.linkedin.com/v1/people/~?format=json
   
   sample api response
   ```
   {
     "firstName": "Frodo",
     "headline": "2nd Generation Adventurer",
     "id": "1R2RtA",
     "lastName": "Baggins",
     "siteStandardProfileRequest": {
       "url": "https://www.linkedin.com/profile/view?id=…"
     }
   }
   ```
 

