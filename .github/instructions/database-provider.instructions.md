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
| **Filter** | Props nullable (`DateTime?`, `int?`) | Pattern: `From{X}`, `To{X}`, `{X}Contains` |
| **DTO** | `[JsonPropertyName]` camelCase | Namespace: `{ROOT_NAMESPACE}.Dto` |
| **Projection** | `Expression<Func<T,R>>` con `{ get; }` | Classe: `{Entity}Extensions`, Property: `To{Entity}Result` |
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
├── {Entity}Result.cs
└── {Entity}Extensions.cs
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
    public async Task<List<TResult>> Get{Entity}Async<TResult>(
        {Entity}Filter filter,
        Expression<Func<{Entity}, TResult>> selector,
        CancellationToken cancellationToken)
    {
        var query = context.{Entity}.AsNoTracking();
        
        if (filter.From{Field}.HasValue)
            query = query.Where(e => e.{Field} >= filter.From{Field}.Value);
        
        return await query.Select(selector).ToListAsync(cancellationToken);
    }
}
```

### 3️⃣ Entity
```csharp
[Table("{TABLE_NAME}")]
public class {Entity}
{
    [Column("{ColumnName}")] // OBBLIGATORIO su ogni prop
    public int Id { get; set; }
}
```

### 4️⃣ Filter
```csharp
public class {Entity}Filter
{
    public DateTime? From{Field} { get; set; } // DateTime → From/To
    public DateTime? To{Field} { get; set; }
    public string? {Field}Contains { get; set; } // string → Contains
    public int? {Field} { get; set; } // numeric → stesso nome
}
```

### 5️⃣ DTO Result + Projection Extension
```csharp
// {Entity}Result.cs
public class {Entity}Result
{
    [JsonPropertyName("fieldName")] // camelCase
    public string FieldName { get; set; }
}

// {Entity}Extensions.cs
public static class {Entity}Extensions
{
    public static Expression<Func<{Entity}, {Entity}Result>> To{Entity}Result { get; } = 
        source => new {Entity}Result { FieldName = source.FieldName };
}
```
**⚠️ IMPORTANTE**: Usa `{ get; }` NON `=>` per performance

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

### 8️⃣ Uso negli Endpoint
```csharp
var result = await provider.Get{Entity}Async(
    filter, 
    {Entity}Extensions.To{Entity}Result, // ← Expression statica
    cancellationToken);
```

---

## 📋 Regole Obbligatorie

| Regola | Dettaglio |
|--------|-----------|
| **Primary Constructor** | Sempre per DbContext e Provider |
| **CancellationToken** | Ultimo parametro in TUTTI i metodi che materializzano (ToListAsync, etc) |
| **AsNoTracking()** | Su tutte le query read-only |
| **Logging** | `logger.LogInformation` per ogni filtro applicato |
| **Column Attribute** | Su OGNI proprietà Entity |
| **Filter per Entity** | Ogni Entity DEVE avere il suo Filter (anche vuoto) |
| **Projection Extension** | Ogni Entity DEVE avere Extension + Result DTO |
| **Expression Pattern** | `{ get; }` NON `=>` |
| **Git Commit** | Solo locale, NO push automatico |

---

## 📖 Riferimento: Provider Antifrode Esistente

**Invece di duplicare codice, CONSULTA i file esistenti nel progetto**:
- `src/D106.Api/Infrastructure/Antifrode/` - Implementazione completa
- Entities: `Log106.cs`, `CS2K.cs`
- Filters: `Log106Filter.cs`, `CS2KFilter.cs`
- DTOs: `Dto/Log106Extensions.cs`, `Dto/CS2KExtensions.cs`

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
git checkout -- <file>        # Ripristina file singolo
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

- [ ] Per ogni Entity: Filter + DTO Result + DTO Extensions esistono
- [ ] Filter: DateTime → `From{X}`/`To{X}`, string → `{X}Contains`
- [ ] Projection Extension: `{ get; }` NON `=>`, nome `To{Entity}Result`
- [ ] Provider: metodo `Get{Entity}Async<TResult>` con `CancellationToken`
- [ ] `[Column]` attributo su OGNI proprietà Entity
- [ ] Logging su ogni filtro applicato
- [ ] DI Extension creato: `Add{PROVIDER}Provider`
- [ ] Registrato in `Program.cs` con using
- [ ] Connection string in `appsettings.Development.json`
- [ ] Endpoint usa Expression statica: `{Entity}Extensions.To{Entity}Result`

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
- Le Expression `To{Entity}Result` hanno `{ get; }` e non `=>`?

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

*Template v1.8 - .NET 10 - Token-optimized for AI agents* - Last Update 2026-03-25 — claude-sonnet-4-6
