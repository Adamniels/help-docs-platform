# Memory Context Routing — Refactor Plan

Introduce workflow-type routing so any workflow can get a custom context provider,
with `EfMemoryContextProvider` as the default fallback.

**Scope:** 3 files changed, 1 new file, 0 breaking changes.

---

## Background

Currently `IMemoryContextProvider` has a single implementation — `EfMemoryContextProvider` —
registered directly against the interface. The `MemoryContextRequest` already carries
`WorkflowType`, but it is only used as a scoring input inside the provider, not for routing.

The goal is to insert a thin dispatcher in front of the default so that workflow-specific
providers can be added later without touching any upstream code.

---

## Step 1 — Register `EfMemoryContextProvider` as itself

**File:** `Platform.Infrastructure/Features/Memory/DependencyInjection/MemoryInfrastructureServiceCollectionExtensions.cs`

Add a direct registration for `EfMemoryContextProvider` so the router can take it as a
dependency. Change the interface binding to point at the new router.

```csharp
// Before
.AddScoped<IMemoryContextProvider, EfMemoryContextProvider>()

// After
.AddScoped<EfMemoryContextProvider>()                                // default, injectable by type
.AddScoped<IMemoryContextProvider, RoutingMemoryContextProvider>()   // router owns the interface
```

---

## Step 2 — Create `RoutingMemoryContextProvider`

**File:** `Platform.Infrastructure/Features/Memory/Context/RoutingMemoryContextProvider.cs` *(new)*

A thin dispatcher. No logic lives here — it only reads `WorkflowType` and delegates.
Falls back to `EfMemoryContextProvider` for anything without a specific match.

```csharp
namespace Platform.Infrastructure.Features.Memory.Context;

public sealed class RoutingMemoryContextProvider(
    EfMemoryContextProvider defaultProvider
    // inject workflow-specific providers here as you build them
) : IMemoryContextProvider
{
    public Task<MemoryContextV1Dto> GetContextAsync(
        MemoryContextRequest request,
        CancellationToken ct = default)
    {
        return request.WorkflowType?.Trim().ToLowerInvariant() switch
        {
            // "side_learning" => sideLearningProvider.GetContextAsync(request, ct),
            _ => defaultProvider.GetContextAsync(request, ct),
        };
    }
}
```

---

## Step 3 — Verify nothing upstream changed

These files resolve `IMemoryContextProvider` via DI and are completely unaffected.
Confirm they still compile and behave identically — no edits needed.

- `PostMemoryContextRequestHandler.cs`
- `GetMemoryContextQueryHandler.cs`
- `MemoryContextV1Routes.cs`

---

## How to add a workflow-specific provider later

Three things each time. Everything else stays the same.

**1. Create the provider**

```csharp
public sealed class SideLearningMemoryContextProvider(...) : IMemoryContextProvider
{
    // completely custom logic — hard SQL filters, different scoring,
    // different data sources, whatever the workflow needs
}
```

The new provider can also take `EfMemoryContextProvider` as a dependency and call it
internally if it wants to augment the default rather than replace it entirely.

**2. Register it in DI**

```csharp
.AddScoped<SideLearningMemoryContextProvider>()
```

**3. Add one case to the router**

```csharp
"side_learning" => sideLearningProvider.GetContextAsync(request, ct),
```

---

## Properties

**Non-breaking** — The interface, request contract, and response shape are completely
unchanged. The Python workers, the API route, and the application layer see zero difference.

**Scoped** — All registrations stay scoped, consistent with the existing EF context lifetime.

**Default safe** — Any workflow type without a registered provider falls through to
`EfMemoryContextProvider`. New workflow types added on the Python side will work out of the
box until a custom provider is built for them.
