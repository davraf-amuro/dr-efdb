---
applyTo: "**"
---

# Database Startup Resilience Rules (AI Agent)

Scopo: garantire che una Minimal API .NET resti avviabile e diagnosticabile
quando il database non è raggiungibile all'avvio o a runtime. Applica sempre nei
progetti Minimal API con dipendenza da DB (EF Core) e seed/inizializzazione al boot.

## Principio

Un database irraggiungibile **non deve mai impedire l'avvio dell'API**. L'app parte
in stato degradato, espone lo stato al frontend e permette il retry senza riavvio.

## Regole core (sempre)

1. Seed/inizializzazione DB all'avvio incapsulati in un **servizio singleton** dedicato (es. `DatabaseStartupService`)
2. Il metodo di inizializzazione **cattura ogni eccezione e non rilancia mai** — ritorna `bool`
3. Ogni fallimento è loggato in modo **strutturato** (placeholder, no string interpolation)
4. Lo stato è esposto via `GET /api/v{version}/status` — **sempre 200**, anche a DB down
5. Il retry è esposto via `POST /api/v{version}/status/retry-database` — 200 se pronto, 503 `ProblemDetails` se ancora down
6. Gli endpoint status/retry sono **`AllowAnonymous`** — l'autenticazione dipende dal DB, devono restare raggiungibili a DB down
7. Errori DB **a runtime** gestiti da un `IExceptionHandler` dedicato che risponde **503 `ProblemDetails`**
   - Anche ogni `BackgroundService`/`IHostedService` che tocca il DB cattura le eccezioni e **non rilancia** (il default `BackgroundServiceExceptionBehavior.StopHost` fermerebbe l'intera app): logga e riprova al ciclo successivo, rilancia solo `OperationCanceledException` in shutdown
8. La connessione si verifica con `IDbContextFactory<T>` + `CanConnectAsync` — mai aprire connessioni a mano
9. Il probe live di `GET status` esegue **anche una lettura reale** su una tabella (es. `Roles.AnyAsync`), non solo `CanConnectAsync`: un DB che accetta connessioni ma con file dati illeggibile (es. SQL error 823) deve risultare **non pronto**

## Componenti (responsabilità)

| Componente | Tipo | Responsabilità |
|------------|------|----------------|
| `DatabaseStartupService` | singleton | Stato DB (`IsDatabaseReady`, `LastError`, `LastCheckedUtc`), `TryInitializeAsync` (connessione+seed, no-throw), `CheckConnectionAsync` (probe live) |
| `HealthMapping` | endpoint | `GET status` (live, 200), `POST status/retry-database` (200/503), entrambi `AllowAnonymous` |
| `DatabaseExceptionHandler` | `IExceptionHandler` | Intercetta errori SQL a runtime → 503 `ProblemDetails` |

## Contratto endpoint

| Endpoint | Metodo | Auth | Esito |
|----------|--------|------|-------|
| `api/v{version}/status` | GET | Anonimo | 200 `{ databaseReady, lastError, lastCheckedUtc }` |
| `api/v{version}/status/retry-database` | POST | Anonimo | 200 se DB pronto + seed ok; 503 `ProblemDetails` se ancora down |

## Pattern richiesti (copiabili)

### Servizio singleton — DatabaseStartupService

```csharp
/// <summary>Stato disponibilità DB + inizializzazione resiliente (mai rilancia).</summary>
public class DatabaseStartupService(
    IServiceScopeFactory scopeFactory,
    IDbContextFactory<TContext> contextFactory,
    ILogger<DatabaseStartupService> logger)
{
    public bool IsDatabaseReady { get; private set; }
    public string? LastError { get; private set; }
    public DateTimeOffset? LastCheckedUtc { get; private set; }

    /// <summary>Verifica connessione e, se ok, esegue il seed. Non rilancia mai.</summary>
    public async Task<bool> TryInitializeAsync(CancellationToken ct)
    {
        LastCheckedUtc = DateTimeOffset.UtcNow;
        try
        {
            await using var db = await contextFactory.CreateDbContextAsync(ct);
            if (!await db.Database.CanConnectAsync(ct))
            {
                IsDatabaseReady = false;
                LastError = "Il database non è raggiungibile.";
                logger.LogWarning("Database non raggiungibile durante l'inizializzazione");
                return false;
            }

            // componenti scoped (es. seed ruoli Identity) richiedono uno scope dedicato
            using var scope = scopeFactory.CreateScope();
            var seeder = scope.ServiceProvider.GetRequiredService<TSeedService>();
            await seeder.SeedAsync(ct);

            IsDatabaseReady = true;
            LastError = null;
            return true;
        }
        catch (Exception ex)
        {
            IsDatabaseReady = false;
            LastError = ex.Message;
            logger.LogError(ex, "Errore durante l'inizializzazione del database");
            return false;
        }
    }

    /// <summary>Probe live (per GET status): connessione + lettura reale di una tabella.</summary>
    public async Task<bool> CheckConnectionAsync(CancellationToken ct)
    {
        LastCheckedUtc = DateTimeOffset.UtcNow;
        try
        {
            await using var db = await contextFactory.CreateDbContextAsync(ct);
            if (!await db.Database.CanConnectAsync(ct))
            {
                IsDatabaseReady = false;
                LastError = "Il database non è raggiungibile.";
                return false;
            }

            // lettura reale: CanConnect (SELECT 1) non basta — un DB connettibile ma con
            // file dati illeggibile (SQL error 823) deve risultare non pronto
            await db.Roles.AsNoTracking().AnyAsync(ct);
            IsDatabaseReady = true;
            LastError = null;
            return true;
        }
        catch (Exception ex)
        {
            IsDatabaseReady = false;
            LastError = ex.Message;
            logger.LogWarning(ex, "Verifica connessione database fallita");
            return false;
        }
    }
}
```

### Endpoint status/retry — HealthMapping

```csharp
public static IEndpointRouteBuilder MapHealthEndpoints(this IEndpointRouteBuilder routes, ApiVersionSet versionSet)
{
    var group = routes.MapGroup("api/v{version:apiVersion}/status")
        .WithTags("Status")
        .WithApiVersionSet(versionSet)
        .MapToApiVersion(ApiVersionFactory.Version1)
        .AllowAnonymous();   // auth dipende dal DB → endpoint raggiungibili a DB down

    group.MapGet("/", async (DatabaseStartupService db, CancellationToken ct) =>
    {
        await db.CheckConnectionAsync(ct);
        return TypedResults.Ok(new DatabaseStatusResponse(db.IsDatabaseReady, db.LastError, db.LastCheckedUtc));
    })
    .Produces<DatabaseStatusResponse>(StatusCodes.Status200OK)
    .WithName("GetStatus").WithSummary("Stato del database")
    .WithDescription("Verifica live la raggiungibilità del database.");

    group.MapPost("retry-database", async (DatabaseStartupService db, CancellationToken ct) =>
        await db.TryInitializeAsync(ct)
            ? Results.Ok(new DatabaseStatusResponse(true, null, db.LastCheckedUtc))
            : Results.Problem(new ProblemDetails
            {
                Title = "Base Dati non pronta",
                Detail = db.LastError ?? "Il database non è al momento raggiungibile.",
                Status = StatusCodes.Status503ServiceUnavailable
            }))
    .Produces<DatabaseStatusResponse>(StatusCodes.Status200OK)
    .Produces<ProblemDetails>(StatusCodes.Status503ServiceUnavailable)
    .WithName("RetryDatabase").WithSummary("Ritenta connessione al database")
    .WithDescription("Ritenta connessione + seed; 200 se pronto, 503 se ancora down.");

    return routes;
}

public record DatabaseStatusResponse(bool DatabaseReady, string? LastError, DateTimeOffset? LastCheckedUtc);
```

### Handler errori runtime — DatabaseExceptionHandler

```csharp
/// <summary>Intercetta errori di connessione SQL a runtime → 503 ProblemDetails.</summary>
public class DatabaseExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        if (!IsDbConnectionError(ex)) return false;
        ctx.Response.StatusCode = StatusCodes.Status503ServiceUnavailable;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Title = "Base Dati non pronta",
            Detail = "Il database non è al momento raggiungibile.",
            Status = StatusCodes.Status503ServiceUnavailable
        }, ct);
        return true;
    }

    private static bool IsDbConnectionError(Exception ex) =>
        ex is SqlException ||
        (ex is InvalidOperationException && IsSqlRelated(ex)) ||
        (ex.InnerException is not null && IsDbConnectionError(ex.InnerException));

    private static bool IsSqlRelated(Exception ex) =>
        ex.Message.Contains("sql", StringComparison.OrdinalIgnoreCase) ||
        ex.Message.Contains("connection", StringComparison.OrdinalIgnoreCase) ||
        ex.Message.Contains("database", StringComparison.OrdinalIgnoreCase);
}
```

### Program.cs — registrazione e avvio resiliente

```csharp
builder.Services.AddExceptionHandler<DatabaseExceptionHandler>();
builder.Services.AddSingleton<DatabaseStartupService>();
// ...
app.UseExceptionHandler();
app.MapHealthEndpoints(versionSet);

// SEED resiliente: DB down → l'app parte in stato degradato (no crash)
var dbStartup = app.Services.GetRequiredService<DatabaseStartupService>();
if (!await dbStartup.TryInitializeAsync(CancellationToken.None))
{
    Log.Warning("Database non raggiungibile all'avvio: API in stato degradato. Stato su /api/v1/status, retry su /api/v1/status/retry-database");
}
```

## Perimetro

- NON eseguire seed/inizializzazione DB con codice che può rilanciare nel percorso di avvio
- NON proteggere con auth gli endpoint status/retry (l'auth dipende dal DB)
- NON aprire connessioni manuali per il probe: usa `CanConnectAsync`
- NON duplicare la logica di stato: un solo singleton è la fonte di verità

## Errori comuni (rapidi)

- App crasha all'avvio con DB down → seed non incapsulato nel servizio no-throw
- `Cannot consume scoped service from singleton` → usare `IServiceScopeFactory` per i componenti scoped (seed)
- `GET status` ritorna 503/500 → l'endpoint deve sempre rispondere 200 con `databaseReady=false`
- `GET status` dice `databaseReady=true` ma le query falliscono → probe basato solo su `CanConnectAsync` (SELECT 1): aggiungi una lettura reale (DB connettibile ma file dati corrotto, error 823)
- Background service (`IHostedService`) che usa il DB ferma l'host a DB down → wrappa l'operazione in try/catch no-throw (default `BackgroundServiceExceptionBehavior.StopHost`), rilancia solo su `OperationCanceledException` in shutdown
- Retry inutilizzabile a DB down → endpoint non `AllowAnonymous`
- 500 invece di 503 a runtime → `DatabaseExceptionHandler` non registrato o `UseExceptionHandler()` mancante

## Checklist Post-Generazione

- [ ] `DatabaseStartupService` singleton, `TryInitializeAsync` non rilancia mai
- [ ] Seed scoped risolto via `IServiceScopeFactory`
- [ ] `GET status` sempre 200 con `{ databaseReady, lastError, lastCheckedUtc }`
- [ ] `POST status/retry-database` 200/503 `ProblemDetails`
- [ ] Endpoint status/retry `AllowAnonymous`
- [ ] `DatabaseExceptionHandler` registrato + `UseExceptionHandler()` attivo
- [ ] Avvio con DB down: app parte, log `Warning` strutturato presente

*Template v1.1 - Database Startup Resilience - 2026-06-30 — claude-opus-4-8*
