---
title: "OpenId Connect in ASP.NET MVC with legacy backend"
date: 2019-06-18T10:26:51+05:30
featuredImage: "posts/oidc-in-aspnet-mvc-with-legacy-backend/images/micah-williams-436268-unsplash.jpg"
---

Background:

To integrate [OpenId Connect (henceforth OIDC)](https://openid.net/connect/) in a new ASP.NET MVC application, we can easily use [ASP.NET Identity](https://docs.microsoft.com/en-us/aspnet/identity/) which relies on [OWIN](https://docs.microsoft.com/en-us/aspnet/aspnet/overview/owin-and-katana/).

What to do, when you have a legacy ASP.NET MVC application with custom [Forms Authentication](https://docs.microsoft.com/en-us/aspnet/web-forms/overview/older-versions-security/introduction/an-overview-of-forms-authentication-cs) and identity management taken care by a legacy backend service([SOAP](https://www.w3.org/TR/soap12/#intro))?

I'll try to describe the process of achieving this kind of integration.

### Basics

Before we get to the nuts and bolts of things promised in this post, lets quickly define some key concepts:

OIDC: is an authentication (identity) layer on top of [OAuth 2.0](https://en.wikipedia.org/wiki/OAuth#OAuth_2.0).
The main actors in OIDC are OpenId Provider (OP) and Relying Party (RP). OP is an OAuth 2.0 compliant server, capable of authenticating an user for RP. RP on the other hand is again an OAuth 2.0 application which relies (hence the name) on OP for user authentication.

So, in our case, our classic/legacy web application is the RP and we can use any OP which is OAuth 2.0 compliant! For exploration purposes Auth0, Okta, OneLogin, etc are pretty good implementations of an OP. For this post, I'll be using [okta](https://www.okta.com) OP, you can replace it with an OP of your choice.

OIDC allows three types of flows for authentication and these pretty much dictate how the OP will handle them. Each flow is for a specific use case.
Auth0 and Okta have some fantastic documentation on OIDC in general and in this case of which flow to use in each use case, see [Auth0](https://auth0.com/docs/api-auth/which-oauth-flow-to-use) & [Okta](https://developer.okta.com/docs/concepts/auth-overview/#choosing-an-oauth-2-0-flow).
Following is a list and brief on each flow.

[Authorization Code Flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth): As the name says, it returns an authorization code, that will be used/exchanged for an identity token and/or an access token. Which further requires your client id and secret to request these tokens. This flow is used on Server-side (web applications), as the user performs authentication (via a url redirection), the server application is provided with an authorization code (via a configured callback/redirect url), and the server on a back-channel will communicate with the OP (this is where client id and secret are shared with OP) to authenticate.

[Implicit Flow](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth): This flow, as the name suggests, is used to obtain access token or identity token along with the redirection response from OP. No back-channel request. Mostly used by SPA or mobile apps.

[Hybrid Flow](https://openid.net/specs/openid-connect-core-1_0.html#HybridFlowAuth): This combines the other two flows. It allows use cases where an application can immediately use an identity token for user information and authorization code to be exchanged for access token. Rarely used!

### Back to business

We modified the backend and enabled it to store, 2 additional pieces of data along with a user identity. An OpenId Provider (OP) name and an user identification token. And updated the service interfaces.
With this change, the backend can now allow a login, based on an email, user identification token and OP name, all of which will/can be read from OP.

We'll be using Authorization code flow in this case, as it is a web application.

As we are done with changes to backend, the real work can start in the web application. The first thing we'll do is to enable OP based login; for which we'll update the login UI. Notice the additional "Sign in with Okta" button.
<img src="images/login-with-okta.png" style="width:50%; margin: auto;"/>

We'll need the following nuget packages for the entire OIDC implementation:

* `Install-Package System.IdentityModel.Tokens.Jwt -Version 5.4.0`
* `Install-Package Microsoft.IdentityModel.JsonWebTokens -Version 5.4.0`
* `Install-Package Microsoft.IdentityModel.Logging -Version 5.4.0`
* `Install-Package Microsoft.IdentityModel.Protocols -Version 5.4.0`
* `Install-Package Microsoft.IdentityModel.Protocols.OpenIdConnect -Version 5.4.0`
* `Install-Package Microsoft.IdentityModel.Tokens -Version 5.4.0`

We now need an action, which will be executed, when the user presses the Okta sign in button.

```csharp
[AllowAnonymous]
public ActionResult Authorize()
{
    var queryParams = new NameValueCollection
    {
        {"client_id", settings.ClientId},       //client id provided by OP
        {"redirect_uri", settings.CallbackUrl}, //for OP to return auth code to RP
        {"response_type", "code"},              //authorization code
        {"scope", "openid profile email"}       //openid for id_token
    };

    // sample authorize url can be as follows
    // https://{yourOktaDomain}/oauth2/default/v1/authorize
    var authUrl = settings.AuthorizeUrl;
    var urlBuilder = new UrlBuilder(authUrl)
    {
        Query = ToQueryString(queryParams)
    };

    // The resultant redirect url may look like:
    // https://{yourOktaDomain}/oauth2/default/v1/authorize?client_id=0oabucvy
    // c38HLL1ef0h7&response_type=code&scope=openid&redirect_uri=http%3A%2F%2Flocal
    // host%3A8080%3Acallback
    return Redirect(urlBuilder.ToString());
}

private static string ToQueryString(NameValueCollection paramsCollection)
{
    return string.Join("&",
        paramsCollection.AllKeys.Select(o => 
            $"{HttpUtility.UrlEncode(o)}={HttpUtility.UrlEncode(paramsCollection[o])}"));
}
```

The Okta signin button will `POST` to above `Authorize` MVC action and a redirect to Okta hosted sign-in page is done. Here, the user can supply credentials to authenticate himself/herself.
Upon successful authentication, OP (Okta) will redirect with an authorization code (generally a hash string as a `GET` parameter named as `code`).

Lets see how we'll handle this authorization code response from OP via another MVC action.

```csharp
public async Task<ActionResult> Callback(string code)
{
    var tokenResp = await RequestOpenIdToken(code);
    var idToken = ValidateToken(tokenResp.IdToken, tokenIssuer,
                                settings.ClientId, tokenSigningKeys);

    // Leaving implementation details of "GetUserIdentifier" & "GetUserEmail"
    // for sake of brevity
    var userIdentifierHashOrToken = GetUserIdentifier(idToken);
    var userEmailAddress = GetUserEmail(idToken);

    // Now call our backend service...
    if(await AuthenticationSvc.Login(settings.OidcProviderName,
                                        userIdentifierHashOrToken,
                                        userEmailAddress))
    {
        return RedirectToAction("Secure", "Home");
    }

    return RedirectToAction("Login", "Account");
}

private static readonly HttpClient httpClient = new HttpClient();

private async Task<OpenIdConnectTokenResponse> RequestOpenIdToken(string code)
{
    var requestParams = new Dictionary<string, string>
                        {
                            { "client_id", settings.ClientId },
                            { "client_secret", settings.ClientSecret },
                            { "code", code },
                            { "grant_type", "authorization_code" },
                            { "redirect_uri", settings.CallbackUrl }
                        };

    var postContent = new FormUrlEncodedContent(requestParams);

    var tokenUrl = settings.TokenUrl;

    // This is our back-channel call to OP, notice the "client_secret"
    // in the above params.
    var response = await httpClient.PostAsync(tokenUrl, postContent);
    if (response.StatusCode != HttpStatusCode.OK)
    {
        throw new ExternalAuthenticationException(
            $@"Error retrieving id_token. Status code:
                {response.StatusCode},
                Reason: {response.ReasonPhrase}")
        {
            Detail = JsonConvert.SerializeObject(response, Formatting.Indented)
        };
    }

    // Read the response
    var result = await response.Content.ReadAsStringAsync();

    if(result.IsNullOrEmpty())
        throw new ExternalAuthenticationException($"Empty token response!");

    return JsonConvert.DeserializeObject<OpenIdConnectTokenResponse>(result);
}

private static JwtSecurityToken ValidateToken(
                        string token,
                        string issuer,
                        string audience,
                        IEnumerable<SecurityKey> signingKeys)
{
    if (token.IsNullOrEmpty()) throw new ArgumentNullException(nameof(token));
    if (issuer.IsNullOrEmpty()) throw new ArgumentNullException(nameof(issuer));
    if (signingKeys == null) throw new ArgumentNullException(nameof(signingKeys));
    if (!signingKeys.Any())
        throw new ArgumentException(@"Error: At least one signing key must be
            available to perform token validation",
            nameof(signingKeys));

    var validationParams = new TokenValidationParameters
    {
        RequireExpirationTime = true,
        RequireSignedTokens = true,
        ValidateIssuer = true,
        ValidIssuer = issuer,
        ValidateIssuerSigningKey = true,
        IssuerSigningKeys = signingKeys,
        ValidateLifetime = true,
        ClockSkew = TimeSpan.FromSeconds(10)
    };

    validationParams.ValidateAudience = !audience.IsNullOrEmpty();

    if (validationParams.ValidateAudience)
    {
        validationParams.ValidAudience = audience;
    }

    try
    {
        var result = new JwtSecurityTokenHandler()
            .ValidateToken(token, validationParams, out var validatedToken);
        return (JwtSecurityToken)validatedToken;
    }
    catch (SecurityTokenValidationException stve)
    {
        throw new ExternalAuthenticationException($"Error validating JWT", stve);
    }
}

// Type for handling token response
private class OpenIdConnectTokenResponse
{
    [JsonProperty("access_token")]
    public string AccessToken { get; set; }

    [JsonProperty("token_type")]
    public string TokenType { get; set; }

    [JsonProperty("expires_in")]
    public int ExpiresIn { get; set; }

    [JsonProperty("scope")]
    public string Scope { get; set; }

    [JsonProperty("refresh_token")]
    public string RefreshToken { get; set; }

    [JsonProperty("id_token")]
    public string IdToken { get; set; }
}
```

Summary of what we are doing in the code above:

1. `Callback` action is called via the OP with `code` as a `GET` param  
2. Based on the `code` received, we request an OIDC token from Token endpoint of OP  
3. We validate the `id_token`, see on [okta](https://developer.okta.com/code/dotnet/jwt-validation/) for more details/explaination
4. Then we read user identifier from `id_token`, this can be the `sub` claim or any other specified by OP or if OP allows customization.
5. Read email claim
6. Call our legacy service with this information and upon successful verification, our custom FormsAuthentication should work as is.

Lock image credit: <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px" href="https://unsplash.com/@mr_williams_photography?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Micah Williams"><span style="display:inline-block;padding:2px 3px"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-2px;fill:white" viewBox="0 0 32 32"><title>unsplash-logo</title><path d="M10 9V0h12v9H10zm12 5h10v18H0V14h10v9h12v-9z"></path></svg></span><span style="display:inline-block;padding:2px 3px">Micah Williams</span></a>
