---
applyTo: "**"
---

# DbProvider Template - Infrastructure Provider Generator

Genera provider EF Core per SQL Server seguendo i pattern del progetto D106.

## 🎯 Quick Reference

| Componente | Pattern | Note |
|------------|---------|------|
| **DbContext** | Primary constructor: `(DbContextOptions<T>, ILoggerFactory)` | NoTracking, SensitiveDataLogging |
| **Provider** | Primary constructor: `(TDbContext, ILogger<T>)` | Metodi async con `CancellationToken` |
| **Entity** | `[Table]` + `[Column]` su ogni proprietà | Fluent API per chiavi |
| **Filter** | Props tutte nullable + `ToExpression()` | Pattern: `{X}` (string→Contains, numerici→uguaglianza), `{X}From`/`{X}To` (DateTime) |
| **DTO** | `record` posizionale con `static Projection` | Namespace: `{ROOT_NAMESPACE}.Dto` |
| **Projection** | `static Expression<Func<TEntity,TDto>> Projection` nel record DTO | New-initializer EF-traducibile — niente classi Extensions |
| **DI Extension** | `Add{Provider}Provider(services, config)` | Registra DbContext + Provider come Scoped |

## ⚠️ Workflow Obbligatorio

1. **Verifica esistenza** provider in `Infrastructure\{NOME_PROVIDER}\`
2. **Se NON esiste**, chiedi conferma:
   ```
   ⚠️ Creerò {NOME_PROVIDER}Provider con:
   - N file in Infrastructure\{NOME_PROVIDER}\
   - Modifiche a Program.cs, appsettings.Development.json
   
   Confermi? (Rollback Git disponibile)
   ```
3. **Attendi conferma esplicita** prima di procedere
4. **Dopo conferma**: crea branch Git locale e commit iniziale (NO push)

---

## 📁 Struttura File da Generare

```
Infrastructure/{PROVIDER}/
├── {PROVIDER}DbContext.cs
├── {PROVIDER}Provider.cs
├── {PROVIDER}ProviderExtensions.cs
├── Entities/{Entity}.cs
└── Filters/{Entity}Filter.cs

Dto/
├── {Entity}Dto.cs          ← record completo con static Projection
└── {Entity}SummaryDto.cs   ← record ridotto con static Projection (SELECT ottimizzata)
```

**Prerequisiti NuGet**: Verifica `Microsoft.EntityFrameworkCore.SqlServer` (NO upgrade se presente)

---

## 🔨 Pattern Implementazione

### 1️⃣ DbContext
```csharp
public class {PROVIDER}DbContext(
    DbContextOptions<{PROVIDER}DbContext> options, 
    ILoggerFactory loggerFactory) : DbContext(options)
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured && loggerFactory != null)
        {
            optionsBuilder.UseLoggerFactory(loggerFactory);
            optionsBuilder.EnableSensitiveDataLogging();
            optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information);
        }
    }
    
    public virtual DbSet<{Entity}> {Entity} { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<{Entity}>(e => e.HasKey(...)); // Chiavi
    }
}
```

### 2️⃣ Provider con Expression Selector
```csharp
public class {PROVIDER}Provider(
    {PROVIDER}DbContext context, 
    ILogger<{PROVIDER}Provider> logger)
{
    /// <summary>Reads {Entity} rows matching the filter, projected with the given EF-translatable selector.</summary>
    public async Task<List<TResult>> Get{Entity}Async<TResult>(
        {Entity}Filter filter,
        Expression<Func<{Entity}, TResult>> selector,
        CancellationToken cancellationToken)
    {
        logger.LogInformation("Querying {Entity} with filter {@Filter}", filter);

        // Leggo da DB: WHERE con i soli filtri valorizzati, SELECT con le sole colonne del DTO
        return await context.{Entity}.AsNoTracking()
            .Where(filter.ToExpression())
            .Select(selector)
            .ToListAsync(cancellationToken);
    }

    /// <summary>Updates an existing {Entity}. False if the key does not exist.</summary>
    public async Task<bool> Update{Entity}Async({Entity} entity, CancellationToken cancellationToken)
    {
        // AsTracking: il context è NoTracking di default, senza tracking SaveChanges non rileva le modifiche
        var existing = await context.{Entity}.AsTracking()
            .FirstOrDefaultAsync(e => e.Id == entity.Id, cancellationToken);
        if (existing is null) return false;

        // ... copia campi ...
        await context.SaveChangesAsync(cancellationToken);
        return true;
    }
}
```
Stessa struttura per `Create{Entity}Async` (Add + SaveChanges, Id da IDENTITY) e `Delete{Entity}Async` (`AsTracking` + Remove + SaveChanges).

> **Soft delete non è coperto.** Se richiesto: aggiungi predicato `IsDeleted == false` in `ToExpression()` e fai sì che `DeleteAsync` imposti il flag invece di rimuovere la riga.

### 3️⃣ Entity
```csharp
[Table("{TABLE_NAME}")]
public class {Entity}
{
    [Column("{ColumnName}")] // OBBLIGATORIO su ogni prop
    public int Id { get; set; }
}
```

### 4️⃣ Filter con ToExpression
```csharp
/// <summary>Optional filters for querying {Entity}. All fields nullable — none required.</summary>
public class {Entity}Filter
{
    public int? Id { get; set; }                  // numerici → stesso nome, uguaglianza
    public string? {Field} { get; set; }          // string → stesso nome, match Contains
    public DateTime? {Field}From { get; set; }    // DateTime → coppia {X}From / {X}To
    public DateTime? {Field}To { get; set; }

    /// <summary>Builds the EF-translatable WHERE expression from the populated fields.</summary>
    public Expression<Func<{Entity}, bool>> ToExpression() =>
        e => (Id == null || e.Id == Id)
          && ({Field} == null || e.{Field}.Contains({Field}))
          && ({Field}From == null || e.{Field} >= {Field}From)
          && ({Field}To == null || e.{Field} <= {Field}To);
}
```
Pattern `(campo == null || ...)`: EF parametrizza e ignora i predicati con filtro null — il WHERE contiene solo i filtri valorizzati. Nessun `if` di composizione query nel provider.

### 5️⃣ DTO record con Projection
```csharp
// Dto/{Entity}Dto.cs — record completo
/// <summary>Response DTO for a {Entity} row, with EF-translatable projection.</summary>
public record {Entity}Dto(int Id, string {Field}, DateTime {Field2})
{
    /// <summary>EF-translatable projection (new-initializer, primitive member access only).</summary>
    public static Expression<Func<{Entity}, {Entity}Dto>> Projection =>
        e => new(e.Id, e.{Field}, e.{Field2});
}

// Dto/{Entity}SummaryDto.cs — record ridotto: SELECT con sole colonne necessarie
public record {Entity}SummaryDto(int Id, string {Field})
{
    public static Expression<Func<{Entity}, {Entity}SummaryDto>> Projection =>
        e => new(e.Id, e.{Field});
}
```
**⚠️ IMPORTANTE**: la Projection vive nel record DTO — niente classi `{Entity}Extensions` separate. Solo new-initializer con member access primitivi: metodi extension (`e.ToDto()`) non sono EF-traducibili e causano valutazione client-side.

### 6️⃣ DI Extension
```csharp
public static class {PROVIDER}ProviderExtensions
{
    public static IServiceCollection Add{PROVIDER}Provider(
        this IServiceCollection services, 
        IConfiguration configuration)
    {
        services.AddDbContext<{PROVIDER}DbContext>(options =>
        {
            options.UseSqlServer(configuration.GetConnectionString("{PROVIDER}Db"));
            options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
            options.EnableSensitiveDataLogging();
            options.EnableDetailedErrors(true);
        });
        
        services.AddScoped<{PROVIDER}Provider>();
        return services;
    }
}
```

### 7️⃣ Registrazione Program.cs
```csharp
using {ROOT_NAMESPACE}.Infrastructure.{PROVIDER};
builder.Services.Add{PROVIDER}Provider(builder.Configuration);
```

### 8️⃣ Uso negli Endpoint (tramite Service)
Gli handler non chiamano il provider direttamente: iniettano il Service (`Services/<Entity>Service.cs`, vedi `minimal-api-architecture.instructions.md` regola 12). È il Service a passare la Projection:
```csharp
// Services/<Entity>Service.cs
public Task<List<{Entity}Dto>> GetAllAsync({Entity}Filter filter, CancellationToken cancellationToken) =>
    provider.Get{Entity}Async(filter, {Entity}Dto.Projection, cancellationToken);

public Task<List<{Entity}SummaryDto>> GetSummariesAsync({Entity}Filter filter, CancellationToken cancellationToken) =>
    provider.Get{Entity}Async(filter, {Entity}SummaryDto.Projection, cancellationToken);
```

---

## 📋 Regole Obbligatorie

| Regola | Dettaglio |
|--------|-----------|
| **Primary Constructor** | Sempre per DbContext e Provider |
| **CancellationToken** | Ultimo parametro in TUTTI i metodi che materializzano (ToListAsync, etc) |
| **AsNoTracking()** | Default dal DbContext; letture no-tracking, `AsTracking()` esplicito su Update/Delete |
| **Logging** | `logger.LogInformation` strutturato con i valori del filtro |
| **Column Attribute** | Su OGNI proprietà Entity |
| **Filter per Entity** | Ogni Entity DEVE avere il suo Filter con `ToExpression()` (anche vuoto) |
| **DTO per Entity** | Almeno `{Entity}Dto` completo + `{Entity}SummaryDto` ridotto, ognuno con `static Projection` |
| **Commenti** | `///` su ogni metodo provider/service + inline su operazioni DB (code-organization Regola 6) |
| **Git Commit** | Solo locale, NO push automatico |

---

## 📖 Riferimento: Implementazione ModelKits

**Invece di duplicare codice, CONSULTA l'implementazione di riferimento ModelKits** (progetto test-guideline):
- `src/test-guideline.api/Infrastructure/ModelKits/` — DbContext, Provider CRUD completo, `Entities/ModelKit.cs`, `Filters/ModelKitFilter.cs` con `ToExpression()`
- `src/test-guideline.api/Dto/ModelKitDto.cs` — record con `static Projection`
- `src/test-guideline.api/Services/ModelKitService.cs` — Service che sceglie la Projection e mappa request→entity

**Struttura identica da replicare** per nuovo provider.

---

## 🔐 Git Workflow (Solo Locale)

**⚠️ NO push automatico sul remote**

```bash
# Setup iniziale
git checkout -b feature/add-{provider}-provider
git add .
git commit -m "feat: add {provider} provider infrastructure"

# Rollback se necessario
git status                    # Vedi modifiche
git restore <file>            # Ripristina file singolo (git v2.23+)
git reset --hard HEAD         # Annulla TUTTE le modifiche non committate
git log --oneline             # Storia commit
git reset --hard <hash>       # Torna a commit specifico
```

**L'utente decide autonomamente quando/se fare `git push`**

---

## 🔌 Connection String

```json
{
  "ConnectionStrings": {
    "{PROVIDER}Db": "Server=localhost;Database={DB};Integrated Security=True;TrustServerCertificate=True"
  }
}
```
**Dev**: `appsettings.Development.json` | **Prod**: User Secrets/Azure Key Vault

---

## ✅ Checklist Post-Generazione

- [ ] Per ogni Entity: Filter con `ToExpression()` + `{Entity}Dto` e `{Entity}SummaryDto` con `static Projection`
- [ ] Filter: tutti i campi nullable — string → `{X}` (Contains), numerici → `{X}`, DateTime → `{X}From`/`{X}To`
- [ ] Provider GET: `Where(filter.ToExpression()).Select(selector)` — nessun `if` di composizione query
- [ ] Provider: metodo `Get{Entity}Async<TResult>` con `CancellationToken`; Update/Delete con `AsTracking()`
- [ ] `[Column]` attributo su OGNI proprietà Entity
- [ ] Logging strutturato del filtro
- [ ] `///` su ogni metodo + commento inline su ogni operazione DB
- [ ] DI Extension creato: `Add{PROVIDER}Provider`
- [ ] Registrato in `Program.cs` con using; Service registrato accanto al provider
- [ ] Connection string in `appsettings.Development.json`
- [ ] Handler usano il Service; il Service passa `{Entity}Dto.Projection` al provider

---

## 🎯 Criteri di successo (verificare prima di iniziare)

Prima di iniziare, chiediti:
- [ ] So esattamente quali file creerò/modificherò?
- [ ] Ho verificato che il provider non esista già in `Infrastructure\{NOME_PROVIDER}\`?
- [ ] Ho letto le istruzioni modulari pertinenti?

Se una risposta è NO → chiedi chiarimenti all'utente prima di procedere.

---

## 🔨 Verifica compilazione (obbligatoria)

Dopo aver generato tutti i file, verifica:
- Tutti i `using` necessari sono presenti in ogni file?
- I namespace sono coerenti tra Entity, Filter, DTO, Provider e Extensions?
- I metodi chiamati negli endpoint esistono nel provider con la firma corretta?
- Le `Projection` dei DTO usano solo new-initializer con member access primitivi?

Se non puoi verificare un punto: dichiaralo esplicitamente all'utente.

---

## 🧪 Testing (Opzionale)

**In-Memory DB**:
```csharp
var options = new DbContextOptionsBuilder<TDbContext>()
    .UseInMemoryDatabase(Guid.NewGuid().ToString()).Options;
var context = new TDbContext(options, NullLoggerFactory.Instance);
```
**NuGet**: `Microsoft.EntityFrameworkCore.InMemory` v10.0

---

*Template v1.9 - .NET 10 - Token-optimized for AI agents* - Last Update 2026-06-10 — claude-fable-5
