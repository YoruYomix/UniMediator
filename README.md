# UniMediator

A Unity-specific Mediator pattern library — MediatR's convenience + IL2CPP safety, no DI framework required.

---

## What Can It Do?

UniMediator lets you decouple game logic with a clean request/handler pipeline, with **filters** you can chain in order — perfect for validation, cost deduction, logging, auth, and more.

No DI framework needed. Just `new` up your handlers and filters and register them directly.

---

## Example: Character Enhancement (4★ → 5★)

```
SendAsync(EnhanceRequest)
  → NetworkCheckFilter     (no network → show popup & block)
    → GemCheckFilter       (gem < 400 → show popup & block)
      → GoldCheckFilter    (gold < 10,000 → show popup & block)
        → DeductCostFilter (deduct gems & gold)
          → EnhanceHandler (upgrade 4★→5★, show success popup, return true)
```

### Request / Result

```csharp
public struct EnhanceRequest : IRequest<EnhanceResult>
{
    public int CharacterId;
    public int RequiredGem;
    public int RequiredGold;

    public EnhanceRequest(int characterId, int requiredGem = 400, int requiredGold = 10000)
    {
        CharacterId = characterId;
        RequiredGem = requiredGem;
        RequiredGold = requiredGold;
    }
}

public struct EnhanceResult
{
    public bool Success;
}
```

### Filter 1 — Network Check

```csharp
public class NetworkCheckFilter : IFilter<EnhanceRequest>
{
    public async UniTask InvokeAsync(EnhanceRequest request, CancellationToken ct, Func<UniTask> next)
    {
        if (Application.internetReachability == NetworkReachability.NotReachable)
        {
            // Replace with your own popup call
            Debug.LogWarning("Network is not connected.");
            return; // pipeline blocked — next() is never called
        }
        await next();
    }
}
```

### Filter 2 — Gem Check

```csharp
public class GemCheckFilter : IFilter<EnhanceRequest>
{
    public async UniTask InvokeAsync(EnhanceRequest request, CancellationToken ct, Func<UniTask> next)
    {
        int currentGem = GameData.Inventory.Gem; // replace with your data source
        if (currentGem < request.RequiredGem)
        {
            Debug.LogWarning($"Not enough gems. (Have: {currentGem} / Need: {request.RequiredGem})");
            return;
        }
        await next();
    }
}
```

### Filter 3 — Gold Check

```csharp
public class GoldCheckFilter : IFilter<EnhanceRequest>
{
    public async UniTask InvokeAsync(EnhanceRequest request, CancellationToken ct, Func<UniTask> next)
    {
        int currentGold = GameData.Inventory.Gold;
        if (currentGold < request.RequiredGold)
        {
            Debug.LogWarning($"Not enough gold. (Have: {currentGold} / Need: {request.RequiredGold})");
            return;
        }
        await next();
    }
}
```

### Filter 4 — Deduct Cost

```csharp
public class DeductCostFilter : IFilter<EnhanceRequest>
{
    public async UniTask InvokeAsync(EnhanceRequest request, CancellationToken ct, Func<UniTask> next)
    {
        GameData.Inventory.Gem  -= request.RequiredGem;
        GameData.Inventory.Gold -= request.RequiredGold;

        try
        {
            await next();
        }
        catch
        {
            // Rollback on handler failure
            GameData.Inventory.Gem  += request.RequiredGem;
            GameData.Inventory.Gold += request.RequiredGold;
            throw;
        }
    }
}
```

### Handler

```csharp
[Filter(typeof(NetworkCheckFilter), Order = 0)]
[Filter(typeof(GemCheckFilter),     Order = 1)]
[Filter(typeof(GoldCheckFilter),    Order = 2)]
[Filter(typeof(DeductCostFilter),   Order = 3)]
public class EnhanceHandler : IRequestHandler<EnhanceRequest, EnhanceResult>
{
    public async UniTask<EnhanceResult> HandleAsync(EnhanceRequest request, CancellationToken ct = default)
    {
        // Replace with your actual upgrade logic
        await GameData.Characters.UpgradeStarAsync(request.CharacterId, fromStar: 4, toStar: 5, ct);

        Debug.Log("Enhancement Success! ✨ Upgraded to 5★!");
        return new EnhanceResult { Success = true };
    }
}
```

### Setup & Caller

No DI container needed — just wire everything up with `new`.

```csharp
public class EnhanceButton : MonoBehaviour
{
    private Mediator _mediator;

    private void Awake()
    {
        _mediator = new Mediator();

        // Per-handler filters are declared via [Filter] on the handler class itself.
        // Just register the handler — filters are picked up automatically.
        _mediator.Register<EnhanceRequest, EnhanceResult>(new EnhanceHandler());

        // Optional: global filters apply to every request
        // _mediator.AddFilter(new LoggingFilter());
    }

    public async void OnClickEnhance()
    {
        var result = await _mediator.SendAsync<EnhanceRequest, EnhanceResult>(
            new EnhanceRequest(characterId: 1001));

        // If any filter blocked the pipeline, Success stays false (default)
        if (result.Success)
            Debug.Log("Enhancement complete!");
        else
            Debug.Log("Enhancement was cancelled.");
    }
}
```

| Step | Filter / Handler | Behavior |
|------|-----------------|----------|
| 1 | `NetworkCheckFilter` | No network → log/popup → **block** |
| 2 | `GemCheckFilter` | Gem < 400 → log/popup → **block** |
| 3 | `GoldCheckFilter` | Gold < 10,000 → log/popup → **block** |
| 4 | `DeductCostFilter` | Deduct gem & gold (rollback on failure) |
| 5 | `EnhanceHandler` | Upgrade 4★→5★, success log/popup, **return `true`** |

---

## Example: Weapon Enhancement (Gold only)

```
SendAsync(WeaponEnhanceRequest)
  → NetworkCheckFilter   (no network → block)
    → GoldCheckFilter    (gold < 5,000 → block)
      → DeductGoldFilter (deduct 5,000 gold)
        → WeaponEnhanceHandler (upgrade weapon, return true)
```

```csharp
public struct WeaponEnhanceRequest : IRequest<WeaponEnhanceResult>
{
    public int WeaponId;
    public int RequiredGold;

    public WeaponEnhanceRequest(int weaponId, int requiredGold = 5000)
    {
        WeaponId     = weaponId;
        RequiredGold = requiredGold;
    }
}

public struct WeaponEnhanceResult { public bool Success; }

public class DeductGoldFilter : IFilter<WeaponEnhanceRequest>
{
    public async UniTask InvokeAsync(WeaponEnhanceRequest request, CancellationToken ct, Func<UniTask> next)
    {
        GameData.Inventory.Gold -= request.RequiredGold;
        try   { await next(); }
        catch { GameData.Inventory.Gold += request.RequiredGold; throw; }
    }
}

[Filter(typeof(NetworkCheckFilter), Order = 0)]
[Filter(typeof(GoldCheckFilter),    Order = 1)]
[Filter(typeof(DeductGoldFilter),   Order = 2)]
public class WeaponEnhanceHandler : IRequestHandler<WeaponEnhanceRequest, WeaponEnhanceResult>
{
    public async UniTask<WeaponEnhanceResult> HandleAsync(
        WeaponEnhanceRequest request, CancellationToken ct = default)
    {
        await GameData.Weapons.EnhanceAsync(request.WeaponId, ct);
        Debug.Log("Weapon enhanced successfully! ⚔️");
        return new WeaponEnhanceResult { Success = true };
    }
}
```

```csharp
// Setup & Call
var mediator = new Mediator();
mediator.Register<WeaponEnhanceRequest, WeaponEnhanceResult>(new WeaponEnhanceHandler());

var result = await mediator.SendAsync<WeaponEnhanceRequest, WeaponEnhanceResult>(
    new WeaponEnhanceRequest(weaponId: 2001));
```

---

## Example: Character Awakening (Gems only)

```
SendAsync(AwakenRequest)
  → NetworkCheckFilter  (no network → block)
    → GemCheckFilter    (gem < 800 → block)
      → DeductGemFilter (deduct 800 gems)
        → AwakenHandler (awaken character, return true)
```

```csharp
public struct AwakenRequest : IRequest<AwakenResult>
{
    public int CharacterId;
    public int RequiredGem;

    public AwakenRequest(int characterId, int requiredGem = 800)
    {
        CharacterId = characterId;
        RequiredGem = requiredGem;
    }
}

public struct AwakenResult { public bool Success; }

public class DeductGemFilter : IFilter<AwakenRequest>
{
    public async UniTask InvokeAsync(AwakenRequest request, CancellationToken ct, Func<UniTask> next)
    {
        GameData.Inventory.Gem -= request.RequiredGem;
        try   { await next(); }
        catch { GameData.Inventory.Gem += request.RequiredGem; throw; }
    }
}

[Filter(typeof(NetworkCheckFilter), Order = 0)]
[Filter(typeof(GemCheckFilter),     Order = 1)]
[Filter(typeof(DeductGemFilter),    Order = 2)]
public class AwakenHandler : IRequestHandler<AwakenRequest, AwakenResult>
{
    public async UniTask<AwakenResult> HandleAsync(
        AwakenRequest request, CancellationToken ct = default)
    {
        await GameData.Characters.AwakenAsync(request.CharacterId, ct);
        Debug.Log("Awakening complete! 💎 Character has awakened!");
        return new AwakenResult { Success = true };
    }
}
```

```csharp
// Setup & Call
var mediator = new Mediator();
mediator.Register<AwakenRequest, AwakenResult>(new AwakenHandler());

var result = await mediator.SendAsync<AwakenRequest, AwakenResult>(
    new AwakenRequest(characterId: 1001));
```

---

## API Reference

### Core Interfaces

#### `IRequest` / `IRequest<TResult>`

```csharp
public interface IRequest { }
public interface IRequest<TResult> { }
```

#### `IRequestHandler<TRequest>`

`TRequest` **must be a `struct`** — required for IL2CPP AOT safety.

```csharp
public interface IRequestHandler<TRequest>
    where TRequest : struct, IRequest
{
    UniTask HandleAsync(TRequest request, CancellationToken ct = default);
}
```

#### `IRequestHandler<TRequest, TResult>`

```csharp
public interface IRequestHandler<TRequest, TResult>
    where TRequest : struct, IRequest<TResult>
{
    UniTask<TResult> HandleAsync(TRequest request, CancellationToken ct = default);
}
```

#### `IFilter<TRequest>`

```csharp
public interface IFilter<TRequest> where TRequest : struct, IRequest
{
    UniTask InvokeAsync(TRequest request, CancellationToken ct, Func<UniTask> next);
}
```

---

### `[Filter]` Attribute

Declaratively attach filters to a handler class. Lower `Order` runs first.

```csharp
[Filter(typeof(LoggingFilter))]
[Filter(typeof(AuthFilter), Order = 1)]
public class MyHandler : IRequestHandler<MyRequest> { ... }
```

> **IL2CPP note:** Filter types are read via reflection only once at startup, then cached. No open generics at runtime.

---

### Mediator

```csharp
var mediator = new Mediator();

// Register handler (no return value)
mediator.Register<AttackRequest>(new AttackHandler());

// Register handler (with return value)
mediator.Register<GetPlayerRequest, PlayerData>(new GetPlayerHandler());

// Global filters — applied to every request, in registration order
mediator.AddFilter(new LoggingFilter());
mediator.AddFilter(new ExceptionHandlingFilter());

// Send
await mediator.SendAsync(new AttackRequest(target));
var player = await mediator.SendAsync<GetPlayerRequest, PlayerData>(new GetPlayerRequest(id));

// Disposable registration — unregister when done
var sub = mediator.Register<AttackRequest>(new AttackHandler());
sub.Dispose();
```

---

### Pipeline Execution Order

```
mediator.SendAsync(request)
    → Global filters          (AddFilter registration order)
        → Per-handler filters ([Filter] attribute, ascending Order)
            → Handler.HandleAsync()
        ← Per-handler filters post-process (reverse order)
    ← Global filters post-process (reverse order)
```

Calling `next()` passes control to the next stage. **Not calling `next()` blocks the rest of the pipeline.**

---

### UniTask Support

UniTask is used automatically when `com.cysharp.unitask` is present in the project.
Falls back to `ValueTask` otherwise.

```csharp
#if UNITASK_SUPPORT
public UniTask HandleAsync(AttackRequest request, CancellationToken ct)
    => UniTask.CompletedTask;
#endif
```

---

## Installation

Add via **Window → Package Manager → + → Add package from git URL**:

```
https://github.com/YoruYomix/UniMediator.git?path=com.yoruyomix.UniMediator
```

---

## Package Info

| | |
|---|---|
| Minimum Unity Version | 2021.3 LTS |
| IL2CPP | Supported (Source Generator, no open generics) |
| Dependencies | None (VContainer and UniTask are optional) |
| Distribution | UPM (Unity Package Manager) |

---

## Optional: VContainer Extension

If you're already using [VContainer](https://vcontainer.hadashikick.jp/) as your DI framework, a separate extension package is available.
It automatically scans your assembly and binds all handlers and filters into the container — no manual `Register()` calls needed.

Install via **Window → Package Manager → + → Add package from git URL**:

```
https://github.com/YoruYomix/UniMediator.git?path=com.yoruyomix.UniMediator.VContainer
```

### Setup

```csharp
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // Register Mediator and optional global filters
        builder.UseMediator(mediator =>
        {
            mediator.AddFilter(new LoggingFilter());
            mediator.AddFilter(new ExceptionHandlingFilter());
        });

        // Auto-scan and bind all IRequestHandler<T> implementations in the assembly
        builder.RegisterMediatorHandlers(Assembly.GetExecutingAssembly());
    }
}
```

### Caller

```csharp
public class EnhanceButton : MonoBehaviour
{
    private IMediator _mediator;

    [Inject]
    public void Construct(IMediator mediator) => _mediator = mediator;

    public async void OnClickEnhance()
    {
        var result = await _mediator.SendAsync<EnhanceRequest, EnhanceResult>(
            new EnhanceRequest(characterId: 1001));

        Debug.Log(result.Success ? "Enhancement complete!" : "Enhancement cancelled.");
    }
}
```

With VContainer, all handler and filter dependencies are resolved automatically from the container — no manual wiring required.

---

## License

MIT License
