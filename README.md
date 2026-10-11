# dr-efdb

Pacchetto di linee guida EF Core per backend .NET: provider database su SQL Server e avvio resiliente quando il database non risponde.

## 🧩 Cosa contiene

| File | Tipo | A cosa serve |
|------|------|--------------|
| `.github/instructions/database-provider.instructions.md` | Istruzione | Genera un provider EF Core per SQL Server in `Infrastructure/{PROVIDER}/`: DbContext e Provider con primary constructor, Entity con `[Column]` su ogni proprietà, Filter con `ToExpression()`, DTO `record` con `static Projection`, estensione DI `Add{PROVIDER}Provider`. Chiede conferma prima di creare i file e lavora su un branch Git locale, senza push. |
| `.github/instructions/database-startup-resilience.instructions.md` | Istruzione | Una Minimal API con EF Core deve avviarsi anche a database irraggiungibile, in stato degradato. Impone un `DatabaseStartupService` singleton che non rilancia mai, `GET status` sempre 200, `POST status/retry-database` (200 o 503) anonimi e un `IExceptionHandler` che risponde 503 agli errori SQL a runtime. |

## 🔗 Dipendenze e domini

- Dipende da: nessuna dipendenza dichiarata. Il core `dr-guidelines` arriva comunque per primo se manca dal manifest: è una dipendenza implicita di ogni pacchetto.
- Richiesto da: nessuno. Nessun pacchetto del catalogo dichiara `dr-efdb` tra le sue dipendenze.
- Applicabilità nel catalogo: `appliesTo: dotnet`.
- Dominio del catalogo: nessuno lo elenca. Il dominio `dotnet-backend` elenca solo `dr-dotnet-backend`.
- Tipologie di progetto che lo suggeriscono: `minimal-api` e `worker-service`, come pacchetto opzionale (`optionalPackages`). Nessuna tipologia lo ha in `suggestedPackages`.
- Server MCP suggerito: `db-schema` (`mcpSuggestion` del catalogo). Se tra i pacchetti scelti c'è `dr-efdb`, lo scaffolding (`/dr-scaffold-solution`, `/dr-scaffold-guidelines`) propone di registrarlo in `.mcp.json`. Senza quel server lo scaffolding CRUD chiede a mano i campi dell'entità.
- `dr-minimalapi` **non** è una dipendenza dichiarata, ed è voluto: questo pacchetto si installa sull'intenzione "mi serve un database", che vale anche per un Worker. Il rimando è condizionale al manifest e ha un ripiego, come prescrive `cross-package-references.instructions.md` del core.

File di altri pacchetti citati dalle istruzioni:

| File citato | Chi lo cita | Pacchetto che lo porta |
|-------------|-------------|------------------------|
| `code-organization` (Regola 6, commenti), `cross-package-references.instructions.md` | `database-provider` | `dr-guidelines` (core) |
| `minimal-api-architecture.instructions.md` (regola 12, Service layer) | `database-provider`, rimando condizionale | `dr-minimalapi` |

Pacchetti suggeriti (`suggests` del catalogo): `dr-minimalapi`. L'installer non li installa e non chiede nulla: a fine installazione stampa una riga `[sugg]` con il motivo per ognuno che manca dal manifest.

## 🚀 Come si installa

Di solito non serve farlo a mano. Per una soluzione nuova si segue la guida del core [Creare una soluzione da zero](https://github.com/davraf-amuro/dr-guidelines/blob/main/docs/guida-nuova-soluzione.md): `/dr-scaffold` installa i pacchetti giusti da solo. Il flusso completo non è ancora stato provato sul campo.

A mano. Il core `dr-guidelines` porta `CLAUDE.md`, configurazione e skill: se manca dal manifest, l'installer lo installa per primo, con entrambe le forme qui sotto. Prerequisiti: PowerShell 7, git, `gh auth status` autenticato (l'installer si scarica con `gh api`; i repo sono Public dal 2026-09-21). Esegui dalla **root del repository host**: l'installer usa la cartella corrente come destinazione e non avvisa se sbagli cartella.

L'installer clona `main` da GitHub in una cartella temporanea, copia i file nel progetto host e poi cancella la cartella temporanea.

Via core, un solo installer:

```powershell
Set-Location <root-del-progetto-host>
& ([scriptblock]::Create((gh api repos/davraf-amuro/dr-guidelines/contents/dr-guidelines-install.ps1 -H "Accept: application/vnd.github.raw" | Out-String))) -Package dr-efdb
```

Oppure con l'installer del pacchetto:

```powershell
& ([scriptblock]::Create((gh api repos/davraf-amuro/dr-efdb/contents/dr-efdb-install.ps1 -H "Accept: application/vnd.github.raw" | Out-String)))
```

Nota: la forma breve `irm https://raw.githubusercontent.com/... | iex` è un'alternativa valida, perché i repo sono Public. Su un repo Private risponderebbe 404: lì si usa `gh api`.

## 📦 Cosa finisce nel progetto host

| Percorso nel progetto host | Contenuto |
|----------------------------|-----------|
| `.github/instructions/database-provider.instructions.md` | Copia dell'istruzione sul provider EF Core |
| `.github/instructions/database-startup-resilience.instructions.md` | Copia dell'istruzione sull'avvio resiliente |
| `.ai/dr-guidelines-packages.json` | Voce `dr-efdb` in `installed`: `package`, `installedAt` (data) e `commit` (commit di `main` installato). Il file si crea se manca. |

Il pacchetto non modifica `CLAUDE.md` né i file di configurazione (`.editorconfig`, `.gitignore`, `.gitattributes`, `.claude/settings.json`, `.mcp.json`). Li scrive il core, solo se viene installato insieme perché mancava dal manifest.

Non vengono copiati `README.md`, `LICENSE`, `.github/ISSUE_TEMPLATE/` e l'installer `dr-efdb-install.ps1`.

Senza `-Update` un file già presente nel progetto host viene saltato (`[SKIP]`).

## 🔄 Aggiornare

Tutti i pacchetti del progetto: `/dr-get-latest`.

Solo questo pacchetto: stesso comando dell'installer del pacchetto, con `-Update` in coda:

```powershell
& ([scriptblock]::Create((gh api repos/davraf-amuro/dr-efdb/contents/dr-efdb-install.ps1 -H "Accept: application/vnd.github.raw" | Out-String))) -Update
```

Nota: `-Update` sovrascrive le copie locali. Si installa sempre l'ultimo `main` pushato: una modifica a questo repo non pushata su `main` non arriva nei progetti host.

Un file rimosso da questo repo resta nel progetto host: l'installer copia, non cancella. Fa eccezione un file elencato in `obsoleteArtifacts` del catalogo: `-Update` lo rimuove.

## 🐞 Segnalare un problema o una miglioria

Non correggere la copia nel progetto host: si perde al primo `-Update`.

Dal progetto host usa `/dr-file-feedback <descrizione>` (su Copilot il prompt `.github/prompts/dr-file-feedback.prompt.md` del core). La issue si apre in questo repo, dopo la tua conferma esplicita di titolo e corpo.

| Modello del repo | Quando usarlo |
|------------------|---------------|
| `.github/ISSUE_TEMPLATE/miglioria.md` | Richiesta evolutiva: regola nuova, precisazione, estensione del pacchetto |
| `.github/ISSUE_TEMPLATE/problema.md` | Malfunzionamento: regola sbagliata, ambigua o che porta l'agente fuori strada |

---

*Revisione v1.5 — 2026-10-11 21:00 — claude-opus-5-5*