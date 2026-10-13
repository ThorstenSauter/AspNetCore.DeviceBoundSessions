# ThorstenSauter.AspNetCore.Authentication.DeviceBoundSessions

[Device Bound Session Credentials](https://w3c.github.io/webappsec-dbsc/) (DBSC) for ASP.NET Core cookie
authentication. A supporting browser (Chrome and Edge on Windows today) keeps a key pair in the device's TPM and
proves possession of it whenever the short-lived session cookie is renewed, so a cookie copied off the device
stops working within minutes.

## Why this package exists

Microsoft shipped this implementation as the experimental `Microsoft.AspNetCore.Authentication.DeviceBoundSessions`
`0.11.0-rc.1` package with .NET 11 RC1, then
[removed it before RC2](https://github.com/dotnet/aspnetcore/pull/69462) to rework it, possibly for .NET 12
([dotnet/aspnetcore#66478](https://github.com/dotnet/aspnetcore/issues/66478)). This is a community fork of that
source, kept alive for applications that already depend on it. It is **not** an official Microsoft package.

Once Microsoft ships DBSC support again, migrate to it; this package will be deprecated in its favour.

- Source: [`dotnet/dotnet@3551975`](https://github.com/dotnet/dotnet/tree/3551975be08744f0418857c5bed8ab1545c5dd47/src/aspnetcore/src/Security/Authentication/DeviceBoundSessions),
  the commit the RC1 package was built from, with its test suite.
- License: MIT, original copyright .NET Foundation and Contributors (see [LICENSE](LICENSE)).

### Differences from `0.11.0-rc.1`

- Namespace and package id are `ThorstenSauter.AspNetCore.Authentication.DeviceBoundSessions`; the
  `AddDeviceBoundSession` extension stays in `Microsoft.Extensions.DependencyInjection`.
- No `[Experimental("ASP0031")]` attribute, so no diagnostic to suppress.
- A registration proof with invalid UTF-8 in its `jwk` header returns 400 instead of 500
  ([dotnet/aspnetcore#69180](https://github.com/dotnet/aspnetcore/issues/69180)).

## Usage

```csharp
builder.Services
    .AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie()
    .AddDeviceBoundSession(CookieAuthenticationDefaults.AuthenticationScheme, options =>
    {
        options.ShortLivedCookieExpiration = TimeSpan.FromMinutes(10);
        // Pages on another origin that call this app must be allowed to trigger a refresh.
        options.AllowedRefreshInitiators.Add("www.example.com");
    });
```

The handler answers registration and refresh under `/.well-known/dbsc/`, stores the device's public key in an
encrypted refresh cookie scoped to that path, and swaps the long-lived source cookie for a short-lived session
cookie once a device is bound. Browsers without DBSC support ignore the protocol and keep the source cookie.

Known upstream issues not yet addressed:
[#69618](https://github.com/dotnet/aspnetcore/issues/69618) (session cookie ignores a non-root source cookie path),
[#69619](https://github.com/dotnet/aspnetcore/issues/69619) (derived schemes ignore a per-scheme
`DataProtectionProvider`), [#69620](https://github.com/dotnet/aspnetcore/issues/69620) (source scheme
`ForwardDefaultSelector` not honoured), [#69682](https://github.com/dotnet/aspnetcore/issues/69682) (multiple
DBSC schemes).

## Building

```bash
dotnet test
```

Requires the SDK pinned in `global.json`.
