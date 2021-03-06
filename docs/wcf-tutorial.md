# Using Auth0 with a WCF Service

This tutorial explains how to consume a WCF service and validating the identity of the caller.

In Web Services or APIs there are two ways these services can be called

* Through a client that has access to a key that can be used to obtain a token.
* Through a client that has access to a token that was obtained somehow.

Typically, first scenario usually happens on a trusted client (a script, a desktop application). The second scenario is the browser, a mobile native app.

For this tutorial, we will assume the standard WCF template with a `basicHttpBinding`.

##Making WCF to validate JsonWebTokens generated by Auth0

The integration consists of adding a `ServiceAuthorizationManager` (which is an extensibility point from WCF). This class intercepts all the calls to a specific service and extracts the HTTP `Authorization` header that contains the JsonWebToken. Then it validates the token using a symmetric or asymmetric key, check that it didn't expire and the audience is correct and finally fills a `ClaimsPrincipal` and set it to the thread principal so it can be accessed later.

###1. Install Auth0-WCF-Service-JWT NuGet package

Use the NuGet Package Manager (Tools -> Library Package Manager -> Package Manager Console) to install the **Auth0-MVC** package, running the command:

    Install-Package Auth0-WCF-Service-JWT

> This package creates the `ServiceAuthorizationManager` and will add a set of configuration settings.

###2. Filling Web.Config with your Auth0 settings

The NuGet package also created three empty settings on `<appSettings>`. Replace them with the following settings:

    <appSettings>
      <add key="jwt:SymmetricKey" value="@@account.clientSecret@@" />
      <add key="jwt:AllowedAudience" value="@@account.clientId@@" />
      <add key="jwt:AllowedIssuer" value="https://@@account.namespace@@/" />
    </appSettings>

And make sure to add the `<serviceAuthorization>`

    <serviceAuthorization principalPermissionMode="Custom" serviceAuthorizationManagerType="....ValidateJsonWebToken, ..." />

###3. Accessing user information

Once the user successfully authenticated to the application, a `ClaimsPrincipal` will be generated which can be accessed through the `User` property or `Thread.CurrentPrincipal`

    public class Service1 : IService1
    {
        public string DoWork()
        {
            var claims = (User.Identity as IClaimsIdentity).Claims
            string email = claims.SingleOrDefault(c => c.ClaimType == "email");

            return "Hello from WCF " + Thread.CurrentPrincipal.Identity.Name +  "(" + email + ")";
        }
    }

###4. Attaching a token on the client

Install the NuGet package on the client side

    Install-Package Auth0-WCF-Client

Extract the `id_token` from the `ClaimsPrincipal` and attach it to the WCF request

    // get JsonWebToken from logged in user
    string token = ClaimsPrincipal.Current.FindFirst(c => c.Type == "id_token").Value;
    
    // attach token to WCF request
    client.ChannelFactory.Endpoint.Behaviors.Add(new AttachTokenEndpointBehavior(token));
    
    // call WCF service
    // client.CallService();

> **Note**: the above asumes that the WCF service is protected with the same client secret as the web site. If you want to call a service protected with a different secret you can obtain a delegation token as shown below:
    
    // get JsonWebToken from logged in user
    string token = ClaimsPrincipal.Current.FindFirst(c => c.Type == "id_token").Value;

    // create an Auth0 client to call the /delegation endpoint using the client id and secret of the caller application
    var auth0 = new Auth0.Client("...caller client id...", "...caller client secret...", "@@account.namespace@@");
    var result = auth0.GetDelegationToken(token, "@@account.clientClient@@");
        
    // attach token to WCF request
    client.ChannelFactory.Endpoint.Behaviors.Add(new AttachTokenEndpointBehavior(token));

**Congratulations!**


