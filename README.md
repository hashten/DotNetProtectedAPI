---
title: .NET WebAPI with OpenID Connect
tags: [security]
keywords: OAUTH2, OpenIdConnect, Azure_Cloud_Security
contact: softwaresecurityteam@hm.com
permalink: /pages/security/Authentication_Authorization/.NET/DOTNET-Tutorial-OIDC-WebAPI.html
---

# Introduction

Repo for a sample WebAPI can be found [here](https://dev.azure.com/HM-GROUP/IAO-SoftwareSecurityHub/_git/DotNetProtectedAPI). Using `Microsoft.Identity.Web` NuGet it is so easy to protect your WebApi so this guide will be very short.

## Configure WebAPI Code


## Azure AD configuration
To make the sample in the repo above work we need to make an App Registration in our Azure AD. The application then needs to expose an API which is done by navigating to the App Registration in going to _Expose an API_ and _Add a scope_. Accept the suggested Application ID URI and add a scope following the instructions in the form fields. The application using this API needs to add this API to its API Permissions. For instructions on how to do that, go to the [.NET Web App Tutorial](/pages/security/Authentication_Authorization/.NET/DOTNET-Tutorial-OIDC-WebApp.html).

## Add Roles
If you want to return different information to different groups of users, you need to define appRoles. This is also necessary if you want to use the API from another service, i.e. not delegated access from a user. Also in the App Registrion in Azure Portal, go to _Manifest_ and look for the empty "appRoles" field.
Fill something similar to the following instead, with \<yourRole\> changed to the name of the role that should be granted access to your API. If it is another service that should access on its own, change "allowedMemberTypes" to "App".

```json
	"appRoles": [
		{
			"allowedMemberTypes": [
				"User"
			],
			"description": "Apps that have this role have the ability to invoke my API",
			"displayName": "Can invoke my backend API",
			"id": "fc803414-3c61-4ebc-a5e5-cd1675c14bba",
			"isEnabled": true,
			"lang": null,
			"origin": "Application",
			"value": "<yourRole>"
		}
	]
```
This user role has to be assigned to the User that will call the API. Do this by going to the overview page of the App Registration and among the Application ID etc go to the link below _Managed application in local directory_. There go to _Users and Groups_ and _Add User_. The AppRoles that you defined in the manifest should be choosable when you have chosen a user or group.

When this is done, it is advisable to restrict all other users to acquiring any token to your app. In the same _Enterprise Application_ view as you configuredf _Users and Groups_, go to _Properties_ and switch _User Assignment_ to on.

{% include tip.html content="Read more about `Microsoft.Identity.Web` NuGet in [the wiki](https://github.com/AzureAD/microsoft-identity-web/wiki/web-apis) and look at other samples in the [Azure-Samples repo](https://github.com/Azure-Samples/active-directory-aspnetcore-webapp-openidconnect-v2/)." %}

