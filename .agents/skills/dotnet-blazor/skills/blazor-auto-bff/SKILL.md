---
license: MIT
name: blazor-auto-bff
description: >
  Scaffold and wire a Blazor Web App with Interactive Auto render mode plus a
  server-side Backend-for-Frontend (BFF): OIDC code flow on the server, tokens
  kept server-side (never in the browser), YARP forwarding of /api/* with Bearer
  injection, and dual service registration (server + .Client WASM implementations
  of the same interfaces with different BaseAddress).
  USE FOR: new Auto-mode apps needing organizational auth (OIDC/Entra/OpenIddict),
  adding BFF endpoints to an existing Blazor Web App, wiring dual client/server
  service implementations, fixing "token in browser" or "policy not found" issues
  in Auto apps.
  DO NOT USE FOR: plain scaffolding without auth (use create-blazor-project),
  `-au Individual` Identity UI (template default), single-mode Server/WASM apps,
  component authoring (use author-component), or API fetching patterns in general
  (use fetch-and-send-data).
---

# Blazor Auto + BFF

Reference implementation:server project (BFF role) + client project `.Client` (WASM). Cite it
when a decision below needs a proven example; adapt names, never copy blindly.

## 1. When This Skill Applies (decision)

| Situation | Action |
|---|---|
| `-int Auto` + OIDC/Entra/OpenIddict + tokens must not live in the browser | **This skill.** Server holds the code flow; browser keeps only the session cookie. |
| `-int Auto` anonymous or `-au Individual` Identity sufficient | `create-blazor-project` only. Identity pages stay static SSR. |
| Existing Auto app, plain `HttpClient` to a public API, no login | `fetch-and-send-data`. Do not retrofit BFF without an auth requirement. |
| Standalone WASM SPA (`blazorwasm`) | Out of scope. Re-scaffold with the `blazor` template (`-int WebAssembly` or `-int Auto`). |

Ask before scaffolding when unclear: identity provider (OIDC authority URL, client id, scopes incl. `offline_access` for refresh), downstream API base URL (`Api:BaseUrl`), token store (Redis vs in-memory dev), cultures/localization needs.

## 2. Scaffold the Base

```shell
dotnet new blazor -o {AppName} -int Auto -ai
```

Two-project layout (server hosts; interactive components live in `.Client`):

```
{AppName}/                        # Server (SDK Microsoft.NET.Sdk.Web) — BFF + SSR/prerender
├── Components/App.razor
├── Components/Routes.razor
├── Program.cs                    # server wiring (§3)
└── {AppName}.csproj              # ProjectReference → {AppName}.Client

{AppName}.Client/                 # WASM (SDK Microsoft.NET.Sdk.BlazorWebAssembly)
├── Pages/                        # ALL InteractiveAuto pages go HERE
├── Services/                     # *ClientService implementations (§7)
├── Extensions/                   # AddPortalClientServices-style registration
├── Routes.razor                  # AuthorizeRouteView + RedirectToLogin + ErrorBoundary
├── Program.cs                    # client wiring (§6)
└── _Imports.razor
```

Rules: components with `InteractiveAuto` (or `InteractiveWebAssembly`) must live
in `.Client` — they may reference shared contracts but never server-only types
(`DbContext`, server services, `HttpContext`). The server `.csproj` references
the `.Client` project; the client references only shared contracts.

## 3. Server `Program.cs` (canonical order)

```csharp
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents()
    .AddInteractiveWebAssemblyComponents()
    .AddAuthenticationStateSerialization();

builder.Services.AddCascadingAuthenticationState();

// Cookie auth (LoginPath=/Account/Login, LogoutPath=/Account/Logout) +
// authorization policies incl. shared role matrix + FallbackPolicy if required.
builder.AddAuthenticationServerService();
builder.AddAuthorizationServerService();

// OIDC code flow (PKCE S256, scopes openid/email/profile/roles + api + offline_access).
// Disable client-side token storage: tokens live in ITokenStore, not the cookie.
builder.AddIdentityServerOpenIddict(identityAuthority, webClientId);

// Token store: Redis in production (keyed by session id); in-memory only for local dev.
builder.Services.AddRedisTokenStore();

// Server-side service implementations used during SSR/prerender (InteractiveAuto
// first pass renders on the server, so it needs working implementations too).
builder.Services.AddPortalServerServices();   // per-service HttpClient → Api:BaseUrl + auth handler

// YARP forwarder for the BFF proxy (§4).
builder.Services.AddHttpForwarder();

var app = builder.Build();
// ... UseWebAssemblyDebugging (dev) else UseExceptionHandler("/Error") + UseHsts ...
// ... UseStatusCodePagesWithReExecute, UseHttpsRedirection ...
app.UseAuthentication();
app.UseAuthorization();
app.UseAntiforgery();

// Map auth/BFF endpoints BEFORE the Razor fallback so anonymous
// login/logout/callback routes cannot be captured by a protected UI endpoint.
app.MapOidcBffEndpoints(builder.Configuration);

app.MapStaticAssets().AllowAnonymous();
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .AddInteractiveWebAssemblyRenderMode()
    .AddAdditionalAssemblies(typeof({AppName}.Client._Imports).Assembly);
```

`App.razor` global Auto: `<HeadOutlet @rendermode="InteractiveAuto" />` and
`<Routes @rendermode="InteractiveAuto" />` with `blazor.web.js`. Pages additionally
declare `@rendermode InteractiveAuto` per page (keeps intent explicit if the global
directive ever changes).

## 4. BFF Endpoints (server)

Expose exactly these (mapping order matters — before `MapRazorComponents`):

- `GET /Account/Login?returnUrl=` → `Results.Challenge` (OIDC scheme). Validate
  `returnUrl` (`SafeReturnUrl`: must start with single `/`, no `\\`). `AllowAnonymous`.
- `GET,POST /Account/Logout` → remove session from token store → `SignOutAsync`
  (cookie) → OIDC sign-out with `id_token_hint`, `RedirectUri="/"`.
- `GET,POST /callback` → `AuthenticateAsync` (OIDC handler); build a new
  `ClaimsIdentity` copying `NameIdentifier/Name/Email/roles`; mint
  `sessionId = Guid:N` as the `oidc_sid` claim; `SaveAsync(sessionId, access,
  refresh, expiresUtc)`; strip tokens from the cookie properties; `SignIn` the
  cookie identity. `AllowAnonymous`.
- `GET,POST /logout-callback` → redirect to `RedirectUri ?? "/"`. `AllowAnonymous`.
- `MapForwarder("/api/{**catch-all}", apiBaseUrl).RequireAuthorization()` (YARP):
  transform reads `oidc_sid` from `HttpContext.User` → `GetAccessTokenAsync` →
  if expiring soon (30s buffer) refresh via the stored refresh token and `SaveAsync`
  the new tokens → `ProxyRequest.Headers.Authorization = Bearer <token>`.
  `apiBaseUrl = configuration["Api:BaseUrl"]` (e.g. `http://api:8080` in compose).

## 5. Token Store (server-side, keyed by session id)

Reference shape (`BuildingBlocks.ServiceDefaults.TokenStorage.RedisTokenStore`,
backed by `IDistributedCache`/Redis, registered via `AddRedisTokenStore()` which
reads `ConnectionStrings:redis`):

```csharp
SaveAsync(sid, accessToken, refreshToken, expiresUtc, ct)  // TTLs: access ~ expiry, refresh 7 days
GetAccessTokenAsync(sid, ct)
GetRefreshTokenAsync(sid, ct)
GetExpiryAsync(sid, ct)
RemoveAsync(sid, ct)
```

Refresh is the caller's job: when `GetExpiryAsync` shows expiry within ~30s, the
BFF endpoint exchanges `GetRefreshTokenAsync(sid)` via the OIDC client
(`AuthenticateWithRefreshTokenAsync`) and `SaveAsync`es the new tokens. Wrap this
behind your own `ITokenStore` abstraction if you need to swap stores (e.g.
in-memory for local dev) — never ship multi-instance deployments on in-memory.

## 6. Client `Program.cs` (WASM, canonical)

```csharp
var builder = WebAssemblyHostBuilder.CreateDefault(args);

builder.Services.AddLocalization();
builder.Services.AddAuthorizationCore(options => AppAuthorizationPolicies.Register(options));
builder.Services.AddCascadingAuthenticationState();
builder.Services.AddAuthenticationStateDeserialization(); // pair of server Serialization

// Client implementations call the SERVER's /api/* (BFF), not the downstream API:
// BaseAddress = the host serving the WASM app → browser sends the BFF cookie automatically.
builder.Services.AddPortalClientServices(new Uri(builder.HostEnvironment.BaseAddress));
builder.Services.AddIdentityServerUiServices(new Uri(builder.HostEnvironment.BaseAddress));
```

Notes: authorization policies MUST be registered on the client too — pages use
`[Authorize(Policy = "...")]` and the interactive pass resolves the policy
locally, otherwise every protected page throws `AuthorizationPolicy named '...'
was not found`. UI-only services (toast/dialog/format) register on both sides so
prerender works. Culture bootstrap (if localized): read the persisted culture via
JS interop after `Build()` and set `DefaultThreadCurrentCulture/UICulture`
before `RunAsync()`.

## 7. Dual-Service Recipe (same interface, two implementations)

Contracts live in shared/client-referenced code (`Interfaces/`):

```csharp
ITeacherPortalService.GetDashboardAsync(ct)   // example; one interface per portal/feature
```

| Side | Implementation | HttpClient |
|---|---|---|
| Server (`Portals/*ServerService`) | Calls the downstream API directly: `GetFromJsonAsync("/api/v1/portals/teacher")` with `BaseAddress = Api:BaseUrl` + `DelegatingHandler` that reads `oidc_sid` from `HttpContext.User` → token store → `Authorization: Bearer` | `AddHttpClient<I*, *Server>((sp, http) => http.BaseAddress = apiBase).AddHttpMessageHandler<PortalAuthHandler>()` + `AddHttpContextAccessor` |
| Client (`Services/*ClientService`) | Identical URLs/methods, relative: `GetFromJsonAsync("/api/v1/portals/teacher")` with `BaseAddress = host` (cookie auth, no handler) | `AddHttpClient<I*, *Client>(http => http.BaseAddress = baseAddress)` |

`Routes.razor`: `Router` + `AuthorizeRouteView` (`Authorizing` spinner,
`NotAuthorized → RedirectToLogin`) inside `ErrorBoundary`, `FocusOnNavigate`,
`NotFoundPage`. `MainLayout` carries `[Authorize]`; `Pages/_imports.razor` may
carry `[Authorize]` to protect the whole client surface.

## 8. Don'ts

- Don't store access/refresh tokens in the browser (local/session storage) — they
  live in `ITokenStore` server-side; the browser keeps only the session cookie.
- Don't map BFF endpoints after `MapRazorComponents` — the fallback swallows them.
- Don't register authorization policies server-only — duplicate them client-side.
- Don't put `InteractiveAuto` components in the server project — they prerender
  fine but fail after WASM handoff.
- Don't call the downstream API directly from WASM — always through the BFF
  `/api/*` forwarder so the token never leaves the server.
- Don't use `HttpContext` (or any server-only API) in `.Client` code — guard
  environment-specific calls with `RendererInfo` where unavoidable.

## 9. Related Skills

- Scaffolding without auth → `create-blazor-project`.
- Component patterns/lifecycle → `author-component`.
- Forms/validation → `collect-user-input`.
- API fetching/loading/error states/service abstraction → `fetch-and-send-data`.
- Auth state edge cases (`AcceptsInteractiveRouting`, static Identity pages) → `configure-auth`.
- Prerender double-load/flicker → `support-prerendering`.
- Cross-component state/circuit lifetimes → `coordinate-components`.
- JS interop → `use-js-interop`; page decomposition → `plan-ui-change`.

## 10. Verify

1. `dotnet build` (solution; both projects).
2. `dotnet run` in the server project; first load renders via Server, subsequent
   visits boot WASM — exercise both passes.
3. Login → call a protected `/api/v1/*` from WASM → confirm downstream receives
   `Authorization: Bearer` and the browser never sees the access token (devtools).
4. Reload on a protected page (SSR/prerender path) → same data, no policy errors.
5. Logout → session removed from token store; protected routes redirect to login.

## Self-check (run before returning)

- [ ] Server `Program.cs` order: components + serialization → auth/policy → OIDC → token store → server services → forwarder → BFF endpoints before Razor fallback?
- [ ] Every injected service has an implementation registered in BOTH `Program.cs` files (server impl vs client impl, correct `BaseAddress` each)?
- [ ] Policies registered on both sides; `Serialization` (server) paired with `Deserialization` (client)?
- [ ] No tokens in browser, no server-only refs in `.Client`, BFF endpoints anonymous only where required?
