---
name: "dotnet-authentication"
description: "Load before wiring credentials into the Walmart APIs .NET SDK client in C#, or when a call comes back 401/403. Covers HTTP Basic, a Bearer token and an API key in a header: the credential each scheme takes, the WalmartApIsClientOptions property it goes on, and a startup check that fails fast on a missing secret."
---

# Authenticating the Walmart APIs .NET SDK client

This API authenticates with HTTP Basic, a Bearer token and an API key in a header. Each scheme is a **nullable credentials property** on `WalmartApIsClientOptions`; set the ones below, then construct the client (see `dotnet-client-initialization`).

| Scheme | Property on `WalmartApIsClientOptions` | Credential |
| --- | --- | --- |
| HTTP Basic | `BasicAuth` | `BasicAuthCredentials` |
| Bearer token | `BearerAuth` | `string` |
| API key — `WM_SEC.ACCESS_TOKEN` header | `AccessToken` | `string` |

**That table is the entire authentication surface of this SDK.** `WalmartApIs.Core.Authentication` also holds scheme classes this API does not use, so read what is configurable off the properties above rather than off that folder.

## Basic auth

```csharp
using WalmartApIs.Core.Authentication.Basic;

options.BasicAuth = new BasicAuthCredentials
{
    Username = "...",
    Password = "..."
};
```

Sends `Authorization: Basic base64(username:password)`. Both halves are required — a username with an empty password is misconfigured, not partially configured.

## Bearer token

**API Provider Notes:**

> Use the access token obtained from the Token API.
> Include it in the WM_SEC.ACCESS_TOKEN header.

`options.BearerAuth` is a plain `string` — assign the access token itself, with no `Bearer` prefix, which the SDK adds:

```csharp
options.BearerAuth = "ACCESS_TOKEN";
```

Sends `Authorization: Bearer ACCESS_TOKEN`.

## API key (header)

**API Provider Notes:**

> Access token required for authenticating API requests.
> 
> To obtain a token, call `POST https://marketplace.walmartapis.com/v3/token` with your client ID and client secret using the client credentials grant type. Refer to the [Token API documentation](https://developer.walmart.com/us-marketplace/docs/get-an-access-token) for details.
> 
> Pass the resulting token in the `WM_SEC.ACCESS_TOKEN` header with every API request., Access token required for authenticating API requests.
> 
> To obtain a token, call `POST https://marketplace.walmartapis.com/v3/token` with your client ID and client secret using the OAuth 2.0 client credentials grant type. Refer to the [Token API documentation](https://developer.walmart.com/us-marketplace/docs/get-an-access-token) for details.
> 
> Pass the resulting token in the `WM_SEC.ACCESS_TOKEN` header with every API request., Access token obtained from the Token API. This is required for authenticating requests to Walmart Marketplace APIs., Access token required for authenticating API requests.
> 
> To obtain a token, call `POST https://marketplace.walmartapis.com/v3/token` with your client ID and client secret using the client credentials grant type. Refer to the [Token API documentation](https://developer.walmart.com/us-marketplace/docs/get-an-access-token) for details.
> 
> Pass the resulting token in the `WM_SEC.ACCESS_TOKEN` header with every API request., Access token from POST /v3/token (client_credentials grant).

The key travels in the `WM_SEC.ACCESS_TOKEN` request header. The header name is fixed — you supply only the value:

```csharp
options.AccessToken = "API_KEY";
```

## Combining the schemes

This SDK has more than one credentials property, and its operations compose them; you choose by which properties on `WalmartApIsClientOptions` you set.

## Operations that need no credentials

Some operations on this API are public: they send no credential, whatever you set on `WalmartApIsClientOptions`. You still construct the same client — there is no separate unauthenticated client — and the credentials you configured are simply not applied to those calls.

## ⚠ Missing credentials must stop the app from starting

⚠ **Configure nothing and the request goes out unauthenticated — no exception.** An unset credentials property on `WalmartApIsClientOptions` sends the call with no credential, and it comes back `401`; the SDK never reports the missing credential. A required credential that is not configured is a deployment fault, not a request fault.

**Validate at startup and refuse to boot.** Bind the credentials into your own options type and make the host check it before the app serves anything:

```csharp
builder.Services
    .AddOptions<WalmartApIsClientSettings>()
    .Bind(builder.Configuration.GetSection("WalmartApIsClient"))
    .ValidateDataAnnotations()      // [Required] on each credential property
    .ValidateOnStart();             // throws during startup, not on first request
```

`ValidateOnStart()` is the load-bearing call — without it, `IOptions<T>` validation is lazy and fires on first resolution, which is a request, which is exactly the late failure you are trying to avoid. For a console app, an explicit guard is equally acceptable as long as it runs **before** the app is ready:

```csharp
if (string.IsNullOrWhiteSpace(settings.Username))
    throw new InvalidOperationException(
        "WalmartApIsClient:Username is not configured. Set it via environment variable, " +
        "user-secrets, or your secret store before starting the app.");
```

Three rules for the message it fails with:

- **Name the missing config key**, so the operator knows what to set — `"WalmartApIsClient:Username is not configured"`, not `"authentication failed"`.
- **Never echo the value**, present or absent — no length, no prefix, no masked form. A "configured: AC1234…" line is a secret in a log.
- **Do not fall back to a default, a placeholder, or an unauthenticated client.** Booting degraded hides the fault and pushes it to the first caller.

Check every value the schemes actually need. For this SDK that is:

| Value to validate | Feeds `WalmartApIsClientOptions` property |
| --- | --- |
| `Username` | `BasicAuth` |
| `Password` | `BasicAuth` |
| `BearerAuth` | `BearerAuth` |
| `AccessToken` | `AccessToken` |

## Notes

- Set credentials **before** constructing the client, or inside the `AddWalmartApIsClient(options => ...)` callback when registering via DI. `WalmartApIsClientOptions` is read at construction; mutating it afterwards does not re-wire an existing client.
- Keep secrets out of source — load them from configuration (environment variables, a secret store, or any other `IConfiguration` source) instead of hardcoding, either inside the `AddWalmartApIsClient(options => ...)` callback for a host or via a `ConfigurationBuilder()...Build()` chain for a console app.

