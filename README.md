# Italy.Core.PA

[![NuGet](https://img.shields.io/nuget/v/Italy.Core.PA)](https://www.nuget.org/packages/Italy.Core.PA)
[![.NET](https://img.shields.io/badge/.NET-8.0%20%7C%20Framework%204.8-blueviolet)](https://dotnet.microsoft.com)
[![Licenza](https://img.shields.io/badge/licenza-MIT-green)](LICENSE)

> Spinoff di [Italy.Core](https://github.com/N0T-A-NUMB3R/Italy.Core) — catalogo delle Pubbliche Amministrazioni italiane in tempo reale.

---

## Cos'è

**Italy.Core.PA** è un pacchetto autonomo che aggiunge l'accesso live al catalogo **IndicePA** (Indice delle Pubbliche Amministrazioni), la fonte ufficiale italiana per:

- Dati anagrafici di tutte le PA italiane (Ministeri, Regioni, Comuni, ASL, Università...)
- **Codici SdI** (destinatario fattura elettronica B2G) per ogni ufficio PA
- **PEC ufficiali** per comunicazioni certificate
- Codici fiscali e siti istituzionali verificati

> **Nessuna API key richiesta.** IndicePA espone un'Open Data API pubblica (CKAN DataStore).
> Licenza dati: **CC BY 4.0**.
> Fonte: `indicepa.gov.it/ipa-dati/api/3/action/`

La differenza rispetto a Italy.Core:

| | Italy.Core | Italy.Core.PA |
|---|---|---|
| **Dati** | Embedded SQLite | Live via HTTP (IndicePA) |
| **Aggiornamento** | Con ogni release NuGet | Ogni chiamata API |
| **Offline** | Sempre disponibile | Richiede connessione |
| **Latenza** | Zero (in-process) | ~200–500ms per chiamata |

---

## Installazione

```bash
dotnet add package Italy.Core.PA
```

Richiede `Italy.Core` come dipendenza (installata automaticamente da NuGet).

---

## Avvio rapido

```csharp
var atlante = new Atlante();
var pa      = new ServiziPAEstesi(atlante.Comuni);

// Cerca enti per nome
var enti = await pa.CercaEnteIPAAsync("Comune di Milano");
Console.WriteLine(enti[0].Denominazione);  // "Comune di Milano"
Console.WriteLine(enti[0].PEC);            // "protocollo@pec.comune.milano.it"
Console.WriteLine(enti[0].CodiceIPA);      // "c_f205"

// Codici SdI per fatturazione B2G
var codici = await pa.OttieniCodiciSdIAsync("c_f205");
Console.WriteLine(codici[0].Codice);       // "A4707H"
```

### Con ASP.NET Core e IHttpClientFactory

```csharp
// Program.cs
builder.Services.AddItalyCore();
builder.Services.AddHttpClient<ServiziPAEstesi>();
builder.Services.AddScoped(sp =>
    new ServiziPAEstesi(
        sp.GetRequiredService<Atlante>().Comuni,
        sp.GetRequiredService<IHttpClientFactory>().CreateClient(nameof(ServiziPAEstesi))));
```

---

## Funzionalità

### Ricerca enti per nome

```csharp
var enti = await pa.CercaEnteIPAAsync("INPS", maxRisultati: 5);

enti[0].CodiceIPA;      // "inps"
enti[0].Denominazione;  // "Istituto Nazionale della Previdenza Sociale"
enti[0].TipoEnte;       // TipoEnteIPA.EntePrevidenziale
enti[0].CodiceFiscale;  // "80078750587"
enti[0].PEC;            // "inps@postacert.inps.gov.it"
enti[0].SitoWeb;        // "https://www.inps.it"
enti[0].SiglaProvincia; // "RM"
```

### Enti per provincia

```csharp
var enti = await pa.OttieniEntiIPAPerProvinciaAsync("MI", maxRisultati: 50);
```

### Ricerca per codice fiscale

```csharp
var ente = await pa.OttieniEnteIPAPerCFAsync("80078750587");
ente.Denominazione;  // "Istituto Nazionale della Previdenza Sociale"
```

### Ricerca per codice IPA

```csharp
var ente = await pa.OttieniEnteIPAPerCodiceAsync("agenzia_entrate");
ente.Denominazione;  // "Agenzia delle Entrate"
ente.PEC;            // "protocollo.dc.segreteria@pec.agenziaentrate.it"
```

### Codici SdI per fatturazione elettronica B2G

Il **codice SdI** è il codice a 6-7 caratteri da inserire nel campo `CodiceDestinatario` della FatturaPA quando si fattura a una Pubblica Amministrazione.

```csharp
var enti   = await pa.CercaEnteIPAAsync("Comune di Roma", maxRisultati: 1);
var codici = await pa.OttieniCodiciSdIAsync(enti[0].CodiceIPA);

foreach (var sdi in codici)
{
    Console.WriteLine(sdi.Codice);               // "LM46ST"
    Console.WriteLine(sdi.DenominazioneUfficio); // "Roma Capitale — Ufficio Protocollo"
    Console.WriteLine(sdi.Attivo);               // true
}
```

---

## Gestione errori

```csharp
try
{
    var enti = await pa.CercaEnteIPAAsync("Comune di Milano");
}
catch (IndicepaException ex)
{
    // Server irraggiungibile, timeout (30s) o risposta non valida
    Console.WriteLine(ex.Message);
}
```

---

## Tipologie ente

| Valore | Descrizione |
|---|---|
| `Ministero` | Ministeri della Repubblica |
| `AgenziaGoverniativa` | Agenzie governative (Agenzia Entrate, ecc.) |
| `EntePrevidenziale` | INPS, INAIL, ecc. |
| `Regione` | Regioni e Province Autonome |
| `Provincia` | Province e Città Metropolitane |
| `Comune` | Comuni italiani |
| `UnioneComuni` | Unioni di Comuni |
| `ComunitaMontana` | Comunità Montane |
| `Universita` | Atenei e alta formazione |
| `ASL` | Aziende Sanitarie Locali |
| `AziendaOspedaliera` | AO, IRCCS, policlinici |
| `ARPA` | Agenzie regionali protezione ambiente |
| `CameraCommercio` | Camere di Commercio |
| `AltroEnte` | Altri enti non classificati |

---

## Struttura

```
Italy.Core.PA/
├── ServiziPAEstesi.cs     # Client IndicePA: ricerca enti, SdI, parsing CKAN JSON
│                          # Domain: EnteIPA, CodiceDestinatarioSdI, TipoEnteIPA...
├── Italy.Core.PA.csproj
└── README.md
```

---

## Compatibilità

| Framework | Linguaggio |
|---|---|
| `.NET 8.0` | C# 12 |
| `.NET Framework 4.8` | C# 7.3 |

---

## Dipendenze

| Pacchetto | net8 | net48 |
|---|---|---|
| `Italy.Core` | ≥1.0.0 | ≥1.0.0 |
| `Microsoft.Extensions.Http` | 8.0.0 | 6.0.0 |
| `System.Text.Json` | 8.0.5 | 6.0.10 |
| `PolySharp` | — | 1.14.1 |

---

## Famiglia Italy.Core

| Pacchetto | Repository | Descrizione |
|---|---|---|
| `Italy.Core` | [GitHub](https://github.com/N0T-A-NUMB3R/Italy.Core) | Libreria base — comuni, CF, IBAN, ATECO, banche, farmacie, rifiuti |
| `Italy.Core.ISTAT` | [GitHub](https://github.com/N0T-A-NUMB3R/Italy.Core.ISTAT) | Statistiche live ISTAT — popolazione, inflazione, PIL, lavoro |
| **`Italy.Core.PA`** | **questo repo** | **Catalogo PA live — IndicePA, SdI B2G, PEC ufficiali** |

---

## Licenza

MIT — vedi [LICENSE](LICENSE).

*Dati IndicePA distribuiti con licenza Creative Commons Attribution 4.0 International (CC BY 4.0).*
