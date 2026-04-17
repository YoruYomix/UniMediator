# UniMediator

A Unity-specific Mediator pattern library — MediatR's convenience + IL2CPP safety, no DI framework required.


---

## Features

- **Middleware pipeline** — attach filter chains to each handler via `[Filter]` attributes
- **Filter reuse** — Cost Interfaces (`IHasRequiredGold`, etc.) let filters work across request types without coupling
- **Automatic rollback** — Deduct filters restore currency on downstream failure
- **Cancellation tracking** — `RequestContext` exposes which filter cancelled the request and why
- **Source Generator** — `[Filter]` attributes are read at build time to generate `FilterRegistry` classes; zero runtime reflection
- **UniTask support** — automatically uses `UniTask` when `com.cysharp.unitask` is present, falls back to `ValueTask`
- **VContainer extension** — auto-scans assemblies to register all handlers and filters; no manual wiring (separate package)

---

## Installation

**Window → Package Manager → + → Add package from git URL**

```
https://github.com/YoruYomix/UniMediator.git?path=com.yoruyomix.UniMediator
```

### VContainer Extension (optional)

```
https://github.com/YoruYomix/UniMediator.git?path=com.yoruyomix.UniMediator.VContainer
```

---

## Quick Start

### 1. Define a Request and Result

```csharp
public struct EnhanceRequest : IRequest<EnhanceResult>, IHasRequiredGold, IHasRequiredGem
{
    public int CharacterId  { get; }
    public int RequiredGold { get; }
    public int RequiredGem  { get; }

    public EnhanceRequest(int characterId, int requiredGem = 400, int requiredGold = 10000)
    {
        CharacterId  = characterId;
        RequiredGem  = requiredGem;
        RequiredGold = requiredGold;
    }
}

public struct EnhanceResult
{
    public bool Success;
}
```

### 2. Write a Handler and Declare Filters

```csharp
[Filter(typeof(NetworkCheckFilter), Order = 0)]
[Filter(typeof(GoldCheckFilter),    Order = 1)]
[Filter(typeof(GemCheckFilter),     Order = 2)]
[Filter(typeof(DeductGoldFilter),   Order = 3)]
[Filter(typeof(DeductGemFilter),    Order = 4)]
public class EnhanceHandler : IRequestHandler<EnhanceRequest, EnhanceResult>
{
    public async ValueTask<EnhanceResult> HandleAsync(EnhanceRequest request, CancellationToken ct = default)
    {
        await GameData.Characters.UpgradeStarAsync(request.CharacterId, fromStar: 4, toStar: 5, ct);
        return new EnhanceResult { Success = true };
    }
}
```

### 3. Build the Mediator and Send

```csharp
private IRequestPublisher _publisher;

private void Awake()
{
    _publisher = new Mediator()
        .AddFilter(new ExceptionHandlingFilter())   // global filter
        .Register<EnhanceRequest, EnhanceResult>(new EnhanceHandler(), EnhanceHandlerFilterRegistry.Filters)
        .BuildPublisher();
}

public async void OnClickEnhance()
{
    var cts     = new CancellationTokenSource();
    var context = new RequestContext();

    EnhanceResult? result = await _publisher.SendAsync<EnhanceRequest, EnhanceResult>(
        new EnhanceRequest(characterId: 1001),
        context,
        cts.Token);

    if (result is { Success: true })
        Debug.Log("Enhancement complete!");
    else if (context.CancelledBy is not null)
        Debug.Log($"[{context.CancelledBy.Name}] {context.CancellationReason}");
}
```

---

## Pipeline Overview

Each request flows through its registered filters in order before reaching the handler.

```
Enhance Pipeline
NetworkCheck(retry×3) → GoldCheck → GemCheck → DeductGold → DeductGem → EnhanceHandler
                             ↓           ↓            ↓            ↓
                     InsufficientGold  InsufficientGem  Rollback  Rollback

Awakening Pipeline
NetworkCheck(retry×3) → GoldCheck → ChaliceCheck → DeductGold → DeductChalice → AwakeningHandler
                             ↓             ↓              ↓              ↓
                     InsufficientGold  InsufficientChalice  Rollback  Rollback
```

---

## Core Concepts

### Cost Interfaces

Separating cost information into interfaces lets filters operate across different request types without knowing the concrete type.

```csharp
public interface IHasRequiredGold    { int RequiredGold    { get; } }
public interface IHasRequiredGem     { int RequiredGem     { get; } }
public interface IHasRequiredChalice { int RequiredChalice { get; } }
```

### RequestContext

When a filter cancels the pipeline it writes its type and reason into `RequestContext`. The caller can inspect this for logging or UI feedback.

```csharp
public class RequestContext
{
    public Type?   CancelledBy        { get; internal set; }
    public string? CancellationReason { get; internal set; }
}
```

### IFilter

```csharp
public interface IFilter
{
    ValueTask InvokeAsync<T>(T request, RequestContext context,
        RequestContinuation<T> next) where T : IRequest;
}
```

Not calling `next` stops the pipeline. Code after `await next(...)` runs on the way back out, making rollback straightforward.

---

## Built-in Filters

### ExceptionHandlingFilter

Catches unhandled exceptions, logs them, and loads the title scene. Re-throws `OperationCanceledException` (Dispose cancellation) without handling. Register as a global filter.

```csharp
public class ExceptionHandlingFilter : IFilter
{
    public async ValueTask InvokeAsync<T>(T request, RequestContext context,
        RequestContinuation<T> next) where T : IRequest
    {
        try
        {
            await next(request, context);
        }
        catch (OperationCanceledException)
        {
            throw;
        }
        catch (Exception e)
        {
            Debug.LogException(e);
            SceneManager.LoadScene("TitleMenu");
        }
    }
}
```

### NetworkCheckFilter

Checks internet reachability. Retries up to 3 times (1 s apart) before showing a popup and cancelling.

```csharp
public class NetworkCheckFilter : IFilter
{
    public async ValueTask InvokeAsync<T>(T request, RequestContext context,
        RequestContinuation<T> next) where T : IRequest
    {
        const int maxRetry = 3;
        for (int i = 0; i < maxRetry; i++)
        {
            if (Application.internetReachability != NetworkReachability.NotReachable)
                break;

            if (i == maxRetry - 1)
            {
                UIManager.ShowPopup("Network is unstable.");
                context.CancelledBy        = typeof(NetworkCheckFilter);
                context.CancellationReason = "Network unreachable";
                return;
            }

            await Task.Delay(1000);
        }

        await next(request, context);
    }
}
```

### GoldCheckFilter / GemCheckFilter / ChaliceCheckFilter

Show a popup and cancel when the player's balance is below the required amount. Work on any request that implements the corresponding Cost Interface.

```csharp
public class GoldCheckFilter : IFilter
{
    public async ValueTask InvokeAsync<T>(T request, RequestContext context,
        RequestContinuation<T> next) where T : IRequest
    {
        if (request is IHasRequiredGold r && GameData.Inventory.Gold < r.RequiredGold)
        {
            UIManager.ShowPopup("Not enough Gold.");
            context.CancelledBy        = typeof(GoldCheckFilter);
            context.CancellationReason = $"Insufficient Gold (have: {GameData.Inventory.Gold} / need: {r.RequiredGold})";
            return;
        }

        await next(request, context);
    }
}
```

### DeductGoldFilter / DeductGemFilter / DeductChaliceFilter

Deduct the required amount before continuing. Automatically roll back on any downstream exception.

```csharp
public class DeductGoldFilter : IFilter
{
    public async ValueTask InvokeAsync<T>(T request, RequestContext context,
        RequestContinuation<T> next) where T : IRequest
    {
        if (request is not IHasRequiredGold r)
        {
            await next(request, context);
            return;
        }

        GameData.Inventory.Gold -= r.RequiredGold;
        try
        {
            await next(request, context);
        }
        catch
        {
            GameData.Inventory.Gold += r.RequiredGold;
            throw;
        }
    }
}
```

---

## Source-Generated FilterRegistry

The Source Generator reads `[Filter]` attributes at build time and emits a `FilterRegistry` class for each handler. No runtime reflection.

```csharp
// <auto-generated/>
public static class EnhanceHandlerFilterRegistry
{
    public static IFilter[] Filters => new IFilter[]
    {
        new NetworkCheckFilter(),
        new GoldCheckFilter(),
        new GemCheckFilter(),
        new DeductGoldFilter(),
        new DeductGemFilter(),
    };
}
```

---

## API Reference

### `IRequestPublisher.SendAsync`

```csharp
ValueTask<TResult?> SendAsync<TRequest, TResult>(
    TRequest request,
    CancellationToken ct = default)
    where TRequest : IRequest<TResult>;

ValueTask<TResult?> SendAsync<TRequest, TResult>(
    TRequest request,
    RequestContext context,
    CancellationToken ct = default)
    where TRequest : IRequest<TResult>;
```

Returns `null` when the pipeline is cancelled by a filter (i.e., `next` was never called through to the handler).

### `IRequestHandler<TRequest, TResult>`

```csharp
public interface IRequestHandler<TRequest, TResult>
    where TRequest : IRequest<TResult>
{
    ValueTask<TResult> HandleAsync(TRequest request, CancellationToken ct = default);
}
```

### `Mediator` (builder)

| Method | Description |
|--------|-------------|
| `AddFilter(IFilter)` | Registers a global filter applied to every request |
| `Register<TRequest, TResult>(handler, filters)` | Registers a handler with its filter array |
| `BuildPublisher()` | Returns an `IRequestPublisher`; throws `ObjectDisposedException` if already disposed |
| `Dispose()` | Cancels all in-flight requests; subsequent calls throw `ObjectDisposedException` |

---

## VContainer Integration

Use the extension package to auto-scan and bind all handlers and filters — no manual `Register()` calls needed.

```csharp
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.UseMediator(mediator =>
        {
            mediator.AddFilter(new ExceptionHandlingFilter());
            mediator.AddFilter(new LoggingFilter());
        });

        // Scans and binds all IRequestHandler<T> in the assembly
        builder.RegisterMediatorHandlers(Assembly.GetExecutingAssembly());
    }
}
```

Inject `IRequestPublisher` wherever you need it — all handler and filter dependencies are resolved by the container automatically.

```csharp
public class CharacterManager : MonoBehaviour
{
    private IRequestPublisher _publisher;

    [Inject]
    public void Construct(IRequestPublisher publisher) => _publisher = publisher;
}
```

---

## License

MIT
