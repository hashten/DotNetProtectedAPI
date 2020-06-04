# DotNetProtectedAPI

For this API to be callable from your web app you need to:
1. Expose an API in the registered App in AzureAD by adding a scope for the registered Web API
2. Give the Web App permission to this scope.


            services.AddAuthorization(options =>
            {
                options.AddPolicy("RequireClaimA", policy => policy.RequireClaim("ClaimA"));
                options.AddPolicy("RequireClaimB", policy => policy.RequireClaim("ClaimB"));
            });

            services.AddMvc(o =>
            {
                o.Filters.Add(new AuthorizeFilter("RequireClaimA"));
            })
            .AddRazorPagesOptions(options =>
            {
                options.Conventions.AllowAnonymousToPage("/AllowAnonymousPageViaConvention");
                options.Conventions.AuthorizePage("/AuthorizePageViaConvention", "RequireClaimB");
            })
            .SetCompatibilityVersion(CompatibilityVersion.Latest);
        }

---
title: Tutorial .NET Spring Boot 2.x OIDC
tags: [security]
keywords: OAUTH2, OpenIdConnect, Azure_Cloud_Security
summary: "DOTNET Tutorial OIDC WebApp Authentication"
contact: softwaresecurityteam@hm.com
permalink: /pages/security/azure_security/Authentication_Authorization/DotNET/DOTNET-Tutorial-OIDC-WebApp.html
---

# Tutorial .NET OIDC

**Prerequisites:** [OAUTH2 in a nutshell](https://www.developer.hmgroup.com/pages/security/azure_security/Authentication_Authorization/OAUTH2.0_in_a_nutshell.html), [Oauth2 Flows (Authorization Code Grant Flow)](https://www.developer.hmgroup.com/pages/security/azure_security/Authentication_Authorization/OAUTH2.0_flows.html#authorization-code-grant-flow)

This tutorial is based on the template projects delivered with the NuGet [Microsoft.Identity.Web](https://github.com/AzureAD/microsoft-identity-web/wiki) and helps you setup the template:
* [Fork Template](#fork-template)
* [App registration](#app-registration)
* [Define access policy](#define-access-policy)
* [Using the tokens](#using-the-tokens)
* [Validate ID tokens](#validate-id-tokens)

It is good to have some knowledge of OAuth2 and Open ID Connect and .NET MVC projects. Open ID Connect is a protocol on top of OAuth2 to provide identity and authentication to your app with the help of an authorization provider. The flow is summarized in the following sequence diagram:
![OIDC auth code grant flow](images/Security/authentication_authorization/Open%20ID%20Connect%20auth%20code%20grant%20flow.png)

More samples using `Microsoft.Identity.Web` can be found at https://github.com/Azure-Samples/active-directory-aspnetcore-webapp-openidconnect-v2

## Fork Template
The templates that use `Microsoft.Identity.Web` are not delivered together with the old template projects so you either have to build the latest project template NuGet yourself following the instructions over here https://github.com/AzureAD/microsoft-identity-web/wiki/web-app-template and create a webapp2, or you can clone this repo which also has pipelines with software security tooling White Source and Coverity.

Fork the template repository https://dev.azure.com/HM-GROUP/IAO-SoftwareSecurityHub/_git/IdentityDotNet. You find fork in the menu that opens with the dots next to the repo title. 

The configuration needs to be changed for the app to run and for that we need to register the app in the AD.

## App Registration
If you do not have the permission to register Apps in your AzureAD you have to ask an admin to do it for you.

For AD authentication to work we need to register the App. Go to azure portal and search for App registrations. Click _New Registration_ and fill in the application name. In the App settings, go to Authentication in the left menu and Add a platform. Choose the _Web_ application and fill the redirect URI and the signout URI. For testing, these should be similar to:
```
https://localhost:5001/signin-oidc
https://localhost:5001/signout-oidc
```

The id-token checkbox must be checked below the Implicit flow header. The OpenID Connect response type that `Microsoft.Identity.Web` uses is `code id_token`, which says that it is the authorization code flow with the addition that the authorization endpoint also returns an id token. 

Go to the overview page of your app in App Registrations and copy the ClientId and TenantId to the appsettings.json file. Then go to Certificates & secrets in the left menu and create a new client secret. This should NOT be stored in the appsettings or in the code. In the development process it should rather be stored with dotnet user-secrets. Replace <ClientSecret> with the secret that you just created in the following:
```bash
dotnet user-secrets init
dotnet user-secrets set "AzureAd:ClientSecret" "<ClientSecret"
```

If you remove the project later, remove the secrets with
```
dotnet user-secrets clear
```

The code described after this section is already in this template repo, so you do not need to add it if you forked this repo. However, it can be a good idea to look through how to define fine grained access policy in the next section.

## Define Access Policy
When working with apps that has both public endpoints and protected endpoints that need authorization, it is common that some endpoints that should be protected end up without protection by mistake. This can happen easily happen if the UI does not show links to the protected endpoints, then you miss to try it. That is why it is important to setup default deny on all endpoint and then specifically say which endpoints should be available. To make the access policy manageable, it is also good to define which pages require authentication in only one place in the code. This template project uses some parts that uses the MVC framework and some parts that uses Razor pages, and they define this in different ways.

Access to Razor Pages can be controlled in detail from the _StartUp.cs_ file and uses so called Conventions. We will require authorization for the whole page except for the landing and privacy pages. 

Go back to the `ConfigureServices` method and change the code after `services.AddRazorPages();` to:
```csharp
services.AddRazorPages().AddRazorPagesOptions(options =>
{
    options.Conventions
        .AuthorizeFolder("/")
        .AllowAnonymousToPage("/Index")
        .AllowAnonymousToPage("/Privacy");
}).AddMvcOptions(options =>
{
    var policy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
    options.Filters.Add(new AuthorizeFilter(policy));
}).AddMicrosoftIdentityUI();
```
The default deny policy was already added for all controllers by the code that was already there after `services.AddRazorPages();`.

The Razor page conventions allow creating exceptions from a harder rule but the opposite is not allowed, so the following would not work:
```csharp
services.AddRazorPages()
  .AddRazorPagesOptions(o =>
    {
      o.Conventions
        .AllowAnonymousToFolder("/")
        .AuthorizePage("/Index");
    });
```

## Using the tokens
In the repo a page has been added - _Profile_ inside the _Pages_ folder. On this page we will extract all information from the id token that we get from the token endpoint in the background. This is not normally information that you would show the user. This is just for demonstration on what information we have. The following code in _Profile.cshtml_ iterates the claims and properties :
```razor
@page
@model ProfileModel
@using Microsoft.AspNetCore.Authentication
@{ 
  ViewData["Title"] = "Profile";
}
<h1>@ViewData["Title"]</h1>

<h2>Claims</h2>

<dl>
  @foreach (var claim in User.Claims)
  {
    <dt>@claim.Type</dt>
    <dd>@claim.Value</dd>
  }
</dl>

<h2>Properties</h2>

<dl>
  @foreach (var prop in (await HttpContext.AuthenticateAsync()).Properties.Items)
  {
    <dt>@prop.Key</dt>
    <dd>@prop.Value</dd>
  }
</dl>
```

and the following code in _Profile.cshtml.cs_ can be used later to acquire more user information with the help of an acquired access token
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.Extensions.Logging;
using Microsoft.Identity.Web;

namespace IdentityDotNet.Pages
{
    public class ProfileModel : PageModel
    {
        private readonly ILogger<ProfileModel> _logger;

        public ProfileModel(ILogger<ProfileModel> logger)
        {
            _logger = logger;
        }

        public void OnGet()
        {
        }
    }
}
```

The following was added to the _Layout.cshtml to show the _Profile_ menu item when the user is authenticated:
```razor
@if (User.Identity.IsAuthenticated) {
    <li class="nav-item">
        <a class="nav-link text-dark" asp-area="" asp-page="/Profile">Profile</a>
    </li>
}
```

## Validate ID Tokens
ID tokens must be validated in the web app. Details can be read in the [OpenID Connect specifications](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation). Fortunately for us, the `Microsoft.Identity.Web` authentication middleware solves the validation for us in the background. The things that should be validated are quite logical:
* Issuer and Signature - Was it the intended OpenID Provider that issued the token?
* Audience - Wat it for us that the token was issued?
* Expiry date - Is the token still valid?