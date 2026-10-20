---
applyTo: "**"
---

# DbProvider Template - Infrastructure Provider Generator

Genera provider EF Core per SQL Server seguendo i pattern del progetto D106.

## 🎯 Quick Reference

| Componente | Pattern | Note |
|------------|---------|------|
| **DbContext** | Primary constructor: `(DbContextOptions<T>, ILoggerFactory)` | NoTracking; `EnableSensitiveDataLogging` solo in Development |
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
   - Modifiche a Program.cs, appsettings.local.json
   
   Confermi?
   ```
3. **Attendi conferma esplicita** prima di procedere
4. **Dopo conferma**: branch e commit seguono il ciclo del core (`plan-tracking.instructions.md`, sezione "Branch di esecuzione"): il branch lo sceglie l'utente nel campo `Branch:` del piano, mai creato in autonomia. NO push automatico

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
            // il logging passa da ILoggerFactory (Serilog): niente LogTo(Console.WriteLine)
            optionsBuilder.UseLoggerFactory(loggerFactory);
        }
    }
    
    public virtual DbSet<{Entity}> {Entity} { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<{Entity}>(e => e.HasKey(...)); // Chiavi
    }
}
```
`EnableSensitiveDataLogging` (valori dei parametri nei log SQL) non sta nel DbContext: si abilita solo in Development, nella DI Extension (pattern 6️⃣).

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
        // il filtro non si logga per intero: può contenere dati sensibili (niente {@Filter})
        logger.LogInformation("Querying {Entity} rows");

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
        IConfiguration configuration,
        IHostEnvironment environment)
    {
        services.AddDbContext<{PROVIDER}DbContext>(options =>
        {
            options.UseSqlServer(configuration.GetConnectionString("{PROVIDER}Db"));
            options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
            // valori dei parametri e dettagli degli errori: solo in Development, mai in produzione
            if (environment.IsDevelopment())
            {
                options.EnableSensitiveDataLogging();
                options.EnableDetailedErrors();
            }
        });
        
        services.AddScoped<{PROVIDER}Provider>();
        return services;
    }
}
```

### 7️⃣ Registrazione Program.cs
```csharp
using {ROOT_NAMESPACE}.Infrastructure.{PROVIDER};
builder.Services.Add{PROVIDER}Provider(builder.Configuration, builder.Environment);
```

### 8️⃣ Uso negli Endpoint (tramite Service)
Gli handler non chiamano il provider direttamente: iniettano il Service (`Services/<Entity>Service.cs`). È il Service a passare la Projection.

> **Dove sta la regola completa.** Controlla `.ai/dr-guidelines-packages.json`: se elenca `dr-minimalapi`, la fonte unica sul Service layer è `minimal-api-architecture.instructions.md`, regola 12 — seguila e dichiaralo nell'output. Se non lo elenca, vale il ripiego: fra chi riceve la richiesta e il provider sta sempre un Service, che riceve il provider per iniezione e non espone mai le entità del database verso l'esterno. Dichiara nell'output che stai applicando il ripiego. Questo pacchetto si installa sull'intenzione "mi serve un database", che vale per una Minimal API quanto per un Worker: non dare per scontato di essere in una API. Se il core `dr-guidelines` è installato, la convenzione che regola questi rimandi è in `cross-package-references.instructions.md`.

Esempio:
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
| **Logging** | `logger.LogInformation` strutturato con placeholder e argomenti coerenti; mai il filtro intero né `{@Filter}` (può contenere dati sensibili); `EnableSensitiveDataLogging` solo in Development |
| **Column Attribute** | Su OGNI proprietà Entity |
| **Filter per Entity** | Ogni Entity DEVE avere il suo Filter con `ToExpression()` (anche vuoto) |
| **DTO per Entity** | Almeno `{Entity}Dto` completo + `{Entity}SummaryDto` ridotto, ognuno con `static Projection` |
| **Commenti** | `///` su ogni metodo provider/service + inline su operazioni DB (code-organization Regola 6) |
| **Git** | Branch e commit secondo il ciclo del core (piano, campo `Branch:`); NO push automatico |

---

## 📖 Esempio di riferimento: ModelKit

I pattern istanziati su un'entità reale: è l'esempio da replicare per un nuovo provider. Il DbContext dell'esempio coincide col pattern 1️⃣; `EnableSensitiveDataLogging` sta nella DI Extension, solo in Development. Provider (`ModelKitsProvider`, metodo `GetModelKitAsync<TResult>`) e DI Extension seguono 2️⃣ e 6️⃣ con `{PROVIDER}` = `ModelKits` e `{Entity}` = `ModelKit`, senza variazioni.

```csharp
// Infrastructure/ModelKits/Entities/ModelKit.cs
[Table("MODEL_KITS")]
public class ModelKit
{
    [Column("ID")] public int Id { get; set; }
    [Column("NAME")] public string Name { get; set; } = "";
    [Column("SCALE")] public string Scale { get; set; } = "";
    [Column("RELEASED_AT")] public DateTime ReleasedAt { get; set; }
}

// Infrastructure/ModelKits/ModelKitsDbContext.cs
public class ModelKitsDbContext(
    DbContextOptions<ModelKitsDbContext> options,
    ILoggerFactory loggerFactory) : DbContext(options)
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured && loggerFactory != null)
        {
            optionsBuilder.UseLoggerFactory(loggerFactory);
        }
    }

    public virtual DbSet<ModelKit> ModelKit { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder) =>
        modelBuilder.Entity<ModelKit>(e => e.HasKey(k => k.Id));
}

// Infrastructure/ModelKits/Filters/ModelKitFilter.cs
/// <summary>Optional filters for querying ModelKit. All fields nullable — none required.</summary>
public class ModelKitFilter
{
    public int? Id { get; set; }
    public string? Name { get; set; }
    public DateTime? ReleasedAtFrom { get; set; }
    public DateTime? ReleasedAtTo { get; set; }

    /// <summary>Builds the EF-translatable WHERE expression from the populated fields.</summary>
    public Expression<Func<ModelKit, bool>> ToExpression() =>
        e => (Id == null || e.Id == Id)
          && (Name == null || e.Name.Contains(Name))
          && (ReleasedAtFrom == null || e.ReleasedAt >= ReleasedAtFrom)
          && (ReleasedAtTo == null || e.ReleasedAt <= ReleasedAtTo);
}

// Dto/ModelKitDto.cs e Dto/ModelKitSummaryDto.cs
public record ModelKitDto(int Id, string Name, string Scale, DateTime ReleasedAt)
{
    public static Expression<Func<ModelKit, ModelKitDto>> Projection =>
        e => new(e.Id, e.Name, e.Scale, e.ReleasedAt);
}

public record ModelKitSummaryDto(int Id, string Name)
{
    public static Expression<Func<ModelKit, ModelKitSummaryDto>> Projection =>
        e => new(e.Id, e.Name);
}

// Services/ModelKitService.cs — sceglie la Projection, non espone l'entità
public class ModelKitService(ModelKitsProvider provider)
{
    /// <summary>Lists model kits matching the filter, as summary rows.</summary>
    public Task<List<ModelKitSummaryDto>> GetSummariesAsync(ModelKitFilter filter, CancellationToken cancellationToken) =>
        provider.GetModelKitAsync(filter, ModelKitSummaryDto.Projection, cancellationToken);
}
```

---

## 🔐 Git

**⚠️ NO push automatico sul remote.** Branch, commit e rollback seguono il ciclo del core `dr-guidelines`: piano su disco e branch di esecuzione scelto dall'utente nel campo `Branch:` (`plan-tracking.instructions.md`, sezione "Branch di esecuzione"). L'agente non crea branch né commit di sua iniziativa e non propone `git reset --hard`: per annullare una modifica indica `git restore <file>` (file singolo) o lascia decidere all'utente.

**L'utente decide quando e se fare `git push`**

---

## 🔌 Connection String

```json
{
  "ConnectionStrings": {
    "{PROVIDER}Db": "Server=localhost;Database={DB};Integrated Security=True;TrustServerCertificate=True"
  }
}
```
**Dev**: `appsettings.local.json` (non committato, caricato in `Program.cs` con `AddJsonFile("appsettings.local.json", optional: true)`) | **Prod**: User Secrets/Azure Key Vault. In `appsettings.json` solo valori fake. Regola completa: `sensitive-data.instructions.md` del core `dr-guidelines`.

---

## 🗄️ Server MCP `db-schema` (se presente)

Se la struttura della tabella non è nota, prima di chiederla all'utente verifica se è disponibile il server MCP `db-schema` (suite `dr-*`, SQL Server): i suoi tool sono esposti nella sessione, oppure è registrato in `.mcp.json`, `.vscode/mcp.json` o nei settings (ricerca completa: `mcp-server-discovery.instructions.md` del core `dr-guidelines`, se installato).

- **Presente**: usa i suoi tool per selezionare la connessione, elencare tabelle/viste e leggere colonne, tipi e nullabilità; da questi ricava `[Table]`/`[Column]`, i campi del Filter e i DTO. Una query di lettura è possibile solo se abilitata nella configurazione del server; le operazioni DDL sono disattivate di default. Dichiara nell'output che la struttura viene dal server.
- **Assente**: se il core è installato, la sua regola propone l'installazione con `setup.ps1` di `davraf-amuro/dr-mcp-dbschema` (solo `win-x64`, SQL Server raggiungibile in rete) e si procede solo dopo conferma dell'utente. Ripiego: chiedi all'utente nome tabella, colonne, tipi e nullabilità e dichiara che la struttura è fornita a mano.

---

## ✅ Checklist Post-Generazione

- [ ] Per ogni Entity: Filter con `ToExpression()` + `{Entity}Dto` e `{Entity}SummaryDto` con `static Projection`
- [ ] Filter: tutti i campi nullable — string → `{X}` (Contains), numerici → `{X}`, DateTime → `{X}From`/`{X}To`
- [ ] Provider GET: `Where(filter.ToExpression()).Select(selector)` — nessun `if` di composizione query
- [ ] Provider: metodo `Get{Entity}Async<TResult>` con `CancellationToken`; Update/Delete con `AsTracking()`
- [ ] `[Column]` attributo su OGNI proprietà Entity
- [ ] Logging strutturato, senza destrutturare il filtro; `EnableSensitiveDataLogging` solo sotto `IsDevelopment()`
- [ ] `///` su ogni metodo + commento inline su ogni operazione DB
- [ ] DI Extension creato: `Add{PROVIDER}Provider`
- [ ] Registrato in `Program.cs` con using; Service registrato accanto al provider
- [ ] Connection string in `appsettings.local.json` (ignorato da git), mai in file committati
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

*Template v1.14 - .NET 10 - Token-optimized for AI agents* - Last Update 2026-10-20 — claude-opus-5-5 — audit suite: logging sensibile solo in Development (DI Extension), niente `LogTo(Console.WriteLine)` né `{@Filter}`, connection string in `appsettings.local.json`, branch/commit dal ciclo del core, sezione server MCP `db-schema`. v1.13 (2026-10-11): rimando a un'implementazione di riferimento inesistente sostituito da un esempio ModelKit inline, DbContext compreso (dr-efdb#1); nota sul logging del DbContext
