# Blazor Identity Scaffolding with Global Interactivity

This document describes the fix for
[dotnet/Scaffolding issue #2694](https://github.com/dotnet/Scaffolding/issues/2694).

## Actual issue

When Blazor Identity is scaffolded into a .NET 9 or .NET 10 Blazor Web App
that uses global Interactive Server rendering, the generated Account pages
inherit the application's unconditional interactive render mode:

```razor
<HeadOutlet @rendermode="InteractiveServer" />
<Routes @rendermode="InteractiveServer" />
```

Identity pages need static server-side rendering because login, registration,
logout, passkey, and other account operations read or write authentication
cookies through the current HTTP request and response. Running those pages in
an interactive circuit can leave `HttpContext` unavailable or attempt to
change a response that has already started.

The generated Account pages opt out of interactive routing with:

```razor
@attribute [ExcludeFromInteractiveRouting]
```

That endpoint metadata is effective only when `App.razor` chooses its render
mode from `HttpContext.AcceptsInteractiveRouting()`.

## Fix implemented

The .NET 9 and .NET 10 `blazorIdentityChanges.json` code-modification configs
now update `Components/App.razor` to use a conditional render mode:

```razor
<HeadOutlet @rendermode="PageRenderMode" />
<Routes @rendermode="PageRenderMode" />

@code {
    [CascadingParameter]
    private HttpContext HttpContext { get; set; } = default!;

    private IComponentRenderMode? PageRenderMode =>
        HttpContext.AcceptsInteractiveRouting() ? InteractiveServer : null;
}
```

The configs also add the endpoint and render-mode imports required by this
code to `Components/_Imports.razor`. Normal pages remain interactive, while
endpoints marked with `[ExcludeFromInteractiveRouting]` render statically.

The fix is intentionally limited to .NET 9 and .NET 10. The .NET 8
scaffolding path has separate existing limitations and no
`blazorIdentityChanges.json` in this repository. Adding only the exclusion
attribute to its Account page imports would not make Login or Register usable,
so that partial change is not included.

No generic `dotnet new` post-processing is used. Blazor Identity's existing
code-modification step owns the host-file changes.

## Test coverage

The existing .NET 9 and .NET 10 Blazor Identity integration tests verify that
the real `blazor-identity` CLI flow:

- creates Login and Register components;
- adds `[ExcludeFromInteractiveRouting]` to the Account page imports;
- changes `HeadOutlet` and `Routes` to use `PageRenderMode`;
- adds the `AcceptsInteractiveRouting()` calculation; and
- produces a project that builds after scaffolding.

The tests use temporary projects and do not depend on a developer-specific
sample directory.

No new standalone test file was added for this fix. The behavior is exercised
by the existing focused integration tests:

- test/dotnet-scaffolding/dotnet-scaffold.Tests/AspNet/Integration/Identity/BlazorIdentityNet9IntegrationTests.cs
- test/dotnet-scaffolding/dotnet-scaffold.Tests/AspNet/Integration/Identity/BlazorIdentityNet10IntegrationTests.cs

These tests seed a globally-interactive `App.razor` and assert that
`Components/App.razor` is updated to use `PageRenderMode`, that `PageRenderMode`
computes `HttpContext.AcceptsInteractiveRouting()`, and that
`Components/Account/Pages/_Imports.razor` contains `@attribute [ExcludeFromInteractiveRouting]`.
If you prefer a faster, focused unit-style test that exercises only the
code-mod replacement (for quicker feedback during development), I can add one
and update this README to document it.

## Test locally

From the repository root, build the tool and run the focused integration
tests:

```powershell
.\build.cmd
dotnet test test\dotnet-scaffolding\dotnet-scaffold.Tests\dotnet-scaffold.Tests.csproj --no-build --filter "FullyQualifiedName~BlazorIdentityNet9IntegrationTests|FullyQualifiedName~BlazorIdentityNet10IntegrationTests"
```

For a manual check, create a .NET 9 or .NET 10 Blazor Web App with no
authentication and global Interactive Server rendering, scaffold Blazor
Identity, and verify:

1. `Components/App.razor` uses `PageRenderMode` for `HeadOutlet` and `Routes`.
2. `PageRenderMode` returns `null` when the endpoint rejects interactive
   routing.
3. `Components/Account/Pages/_Imports.razor` contains
   `[ExcludeFromInteractiveRouting]`.
4. `/Account/Register` and `/Account/Login` render using static SSR.
5. A normal application page still uses Interactive Server rendering.

The render-mode fix does not create an Identity database schema. Before
testing registration, create and apply a migration in the sample application:

```powershell
dotnet ef migrations add CreateIdentitySchema
dotnet ef database update
```
