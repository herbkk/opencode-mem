# OpenCode Memory

> **Nederlandse vertaling.** [English](README.md) is de bronversie; bij verschillen heeft deEngelse versie voorrang. Ververst deze vertaling door de README opnieuw te vertalen bij wijzigingen aan `main`.

[![npm version](https://img.shields.io/npm/v/opencode-mem.svg)](https://www.npmjs.com/package/opencode-mem)
[![npm downloads](https://img.shields.io/npm/dm/opencode-mem.svg)](https://www.npmjs.com/package/opencode-mem)
[![license](https://img.shields.io/npm/l/opencode-mem.svg)](https://www.npmjs.com/package/opencode-mem)

![OpenCode Memory Banner](.github/banner.png)

Een persistent geheugensysteem voor AI-codeeragenten dat langdurige context over sessies heen behoudt, met lokale vectordatabasetechnologie.

## Visueel overzicht

**Tijdlijn van projectgeheugens:**

![Project Memory Timeline](.github/screenshot-project-memory.png)

**Weergave van het gebruikersprofiel:**

![User Profile Viewer](.github/screenshot-user-profile.png)

## Kernfuncties

Lokale Turso/libSQL-database met native vectorzoekopdracht, persistente projectgeheugens, automatische leerfunctie voor het gebruikersprofiel, verenigde geheugen-prompt-tijdlijn, volledig uitgeruste webinterface, intelligente op prompt gebaseerde geheugenextractie, ondersteuning voor meerdere AI-providers (OpenAI, Anthropic), 12+ lokale embeddingmodellen, slimme deduplicatie en ingebouwde privacybescherming.

## Vereisten

Deze plugin gebruikt ingebouwd Turso (`@tursodatabase/database`) met `F32_BLOB`-vectoren en exacte cosinuszoekopdracht via `vector_distance_cos`. Er is geen aparte vectordatabase of aangepaste SQLite-build nodig.

**Aanbevolen runtime:**

- Bun
- De standaard OpenCode-pluginomgeving
- Internettoegang bij het eerste gebruik als je het standaard lokale embeddingmodel gebruikt, omdat het model wordt gedownload door `@huggingface/transformers`.
- Voor installaties vanaf bron/voor ontwikkeling: voer `bun install` uit voor het bouwen of testen. Het gepubliceerde pluginpakket installeert zijn runtime-afhankelijkheden automatisch via OpenCode.

**Platforms getest in CI:** Linux, Windows en macOS 15 / macOS 26 op Apple Silicon (`darwin/arm64`). **Intel Mac (`darwin/x64`) wordt niet ondersteund** — `@tursodatabase/database` en de vaste `onnxruntime-node`-releases bevatten geen x64-native binding. Oudere macOS-versies worden door die matrix niet uitgesloten; ze vallen simpelweg buiten de huidige GitHub-runnerset.

**Opmerkingen:**

- Vector-embeddings worden direct in Turso opgeslagen en doorzocht; bij inserts worden `F32_BLOB`-vectoren bewaard voor exacte cosinusranking.
- Vectorzoekopdracht gebruikt exacte cosinusafstand via `vector_distance_cos` (geen DiskANN / benaderende index).
- Auto-capture en het leren van het gebruikersprofiel vereisen een AI-provider die gestructureerde output of tool-calls kan teruggeven. Zoeken/toevoegen/opsommen van memories werkt ook zonder providerconfiguratie voor auto-capture.

### Upgraden vanaf legacy SQLite-shards

Bij de eerste start na een upgrade migreert opencode-mem de bestaande memory-shard-databases automatisch naar het native Turso/libSQL-vectorkormaat:

- Elke shard wordt vóór het herschrijven geback-upt als `<shard>.db.legacy.bak`
- De voortgang wordt per shard bijgehouden in `<shard>.db.turso-migrate.json`
- De globale marker `.turso-migrated` wordt pas geschreven nadat alle shards succesvol zijn geverifieerd
- Voer tijdens de migratie geen meerdere OpenCode-instanties uit tegen dezelfde `storagePath`; een lockbestand (`.turso-migrate.lock`) voorkomt gelijktijdige migratie
- Handmatige dimensiemigraties gebruiken `.turso-operation.lock`; andere pluginprocessen weigeren nieuwe memory-schrijfacties tot de migratie klaar is

Als de migratie wordt onderbroken, wordt bij de volgende start automatisch hervat vanaf de back-up.

Als een shard incompatibel wordt (bijvoorbeeld na het wijzigen van `embeddingDimensions`), worden schrijfacties geblokkeerd en blijft de originele database ongemoeid. Gebruik de re-embed-migratie van de webinterface om een vervanging te bouwen en te verifiëren voordat die wordt ingewisseld. De vorige shard blijft beschikbaar als `<shard>.db.pre-reembed-<pid>-<timestamp>.bak`.

### Schemamigraties

Lokale Turso-shards en hulpdatabases (`metadata.db`, `user-prompts.db`, `user-profiles.db`, `ai-sessions.db`) worden bijgewerkt met geordende `PRAGMA user_version`-migraties in `src/services/turso/schema-migrations.ts`. Migraties zijn idempotent: bij het starten van de plugin worden alleen openstaande versies toegepast.

## Aan de slag

Voor OpenCode v2 voeg je het pakket toe aan de native `plugins`-lijst:

```jsonc
{
  "plugins": ["opencode-mem@latest"],
}
```

Voor OpenCode v1 voeg je het standaard entrypoint toe aan je configuratie op
`~/.config/opencode/opencode.json`:

```jsonc
{
  "plugin": ["opencode-mem@latest"],
}
```

Met `@latest` (of een semver-bereik) en `autoUpdate: true` in `opencode-mem.jsonc` (de standaard), wist de plugin de gecachte installatie van OpenCode zodra er een nieuwere npm-release beschikbaar is en vraagt hij je om te herstarten. Vastgepinde versies zoals `opencode-mem@2.26.0` worden nooit automatisch bijgewerkt.

### Optionele database-encryptie op schijf

Schakel AES-256-GCM-encryptie in voor lokale Turso-shards in `~/.config/opencode/opencode-mem.jsonc`:

```jsonc
{
  "databaseEncryptionEnabled": true,
}
```

Bij de eerste start maakt de plugin `~/.config/opencode/opencode-mem-db.key` aan (32-byte hex-sleutel, `chmod 600`) en migreert hij bestaande shards in platte tekst. Overschrijf met `"databaseEncryptionKey": "env://OPENCODE_MEM_DB_KEY"` of `"file://~/path/to.key"` als je de sleutel zelf beheert. Verlies je de sleutel, dan kunnen de versleutelde databases niet meer worden geopend.

**Windows:** gebruik `%USERPROFILE%\.config\opencode\opencode.json` (bijvoorbeeld `C:\Users\<jij>\.config\opencode\opencode.json`). Deze plugin leest **niet** `%APPDATA%` of `%LOCALAPPDATA%` voor zijn OpenCode-pluginitem — zet het bestand neer onder `.config\opencode` in je gebruikersprofiel en herstart daarna OpenCode. Verschijnt de plugin niet, controleer dan dat pad en herstart opnieuw.

De plugin wordt bij de volgende start automatisch gedownload.

## Dagelijks gebruik

Je hoeft OpenCode **niet** te vertellen dingen te “onthouden” voor de plugin om te werken. Met de standaardinstellingen bouwt het geheugen zich op terwijl je werkt.

### Typische dagelijkse flow

1. Activeer de plugin (zie [Aan de slag](#aan-de-slag)) en herstart OpenCode.
2. Configureer een AI-provider voor auto-capture — aanbevolen: `opencodeProvider` + `opencodeModel` (of `"opencodeModel": "inherit"`). Details onder [AI-Provider voor auto-capture](#ai-provider-voor-auto-capture).
3. Werk normaal in OpenCode. Wanneer een sessie idle gaat, extraheert auto-capture gedenkwaardige technische context en slaat die op.
4. In latere sessies worden relevante memories in de context geïnjecteerd (zie de instellingen voor `chatMessage` / compaction). Bekijk of bewerk ze in de webinterface op `http://127.0.0.1:4747`.
5. Gebruik de `memory`-tool als je iets direct wilt opslaan of ophalen (zie [Gebruiksvoorbeelden](#gebruiksvoorbeelden)).

### Automatisch versus handmatig geheugen

| Aanpak | Wanneer het draait | Wat je doet |
| --- | --- | --- |
| **Auto-capture** (`autoCaptureEnabled: true`, standaard) | Na gespreksronden zodra de sessie idle gaat | Niets — extractie verloopt automatisch |
| **Handmatig** via de `memory`-tool / commando's | Op aanvraag | `add`, `search`, `list`, `profile`, `forget`, `list-shards`, `migrate`, `export`, `import` |

Handmatig zoeken/toevoegen/opsommen werkt ook als er geen provider is geconfigureerd voor auto-capture. Auto-capture en het leren van het gebruikersprofiel hebben een provider nodig die gestructureerde output of tool-calls kan teruggeven.

### Memory versus AGENTS.md / projectdocumentatie

| Bewaar in **memory** | Bewaar in **AGENTS.md** / statische documentatie |
| --- | --- |
| Projectspecifieke beslissingen, bugpatronen, “we hebben X geprobeerd en het werkte niet” | Stabiele regels en workflows die zelden veranderen |
| Gebruikersvoorkeuren die je over sessies heen ontdekt | Altijd actieve codeconventies en processen |
| Feiten die je over chats heen mee moet nemen | Instructies die elke agent moet zien, ongeacht retrieval |

Vuistregel: is het een blijvende projectinstructie, zet het dan in AGENTS.md; is het context die uit echt werk groeit, laat het dan in memory (of auto-capture).

### Intelligente op prompt gebaseerde geheugenextractie

Die term uit de functielijst is **auto-capture**: na een gesprek vat een achtergrondverzoek aan de AI het technische werk samen en slaat het op als memory. Daarvoor is geen speciale prompt van jou nodig. Het gebruikt `opencodeProvider` / `opencodeModel` als die zijn ingesteld, anders de handmatige `memoryProvider`-fallback.

### Gebruikersprofiel

Het **Gebruikersprofiel** is een aparte, projectoverschrijdende samenvatting van hoe je graag werkt (voorkeuren, gewoonten). Het wordt op een interval bijgewerkt (`userProfileAnalysisInterval`, standaard elke 10 geanalyseerde prompts), getoond in de profielweergave van de webinterface en is uitleesbaar via `memory({ mode: "profile" })`. Voor normaal gebruik vul je het niet handmatig — profielleerwerk vult het zodra er een provider beschikbaar is.

### Webinterface

Open `http://127.0.0.1:4747` om de geheugen-prompt-tijdlijn te bekijken, captures te inspecteren en het gebruikersprofiel te beheren. Als je de server buiten loopback bindt, zie [HTTP Basic Auth voor de webinterface](#http-basic-auth-voor-de-webinterface).

## Gebruiksvoorbeelden

```typescript
memory({ mode: "add", content: "Project uses microservices architecture" });
memory({ mode: "search", query: "architecture decisions" });
memory({ mode: "search", query: "architecture decisions", scope: "all-projects" });
memory({ mode: "profile" });
memory({ mode: "list", limit: 10 });
memory({ mode: "list-shards" });
memory({ mode: "migrate", fromPath: "/old/path/to/project" });
memory({ mode: "export", outputPath: "./memories.json" });
memory({ mode: "import", inputPath: "./memories.json" });
```

Bezoek de webinterface op `http://127.0.0.1:4747` voor visuele verkenning en beheer van memories.

**Beveiliging bij netwerkbinding:** houd `webServerHost` op `127.0.0.1` tenzij je de interface bewust wilt blootstellen. Binding op `0.0.0.0` (of een andere niet-loopback-host) vereist `webServerApiToken`; alle `/api/*`-verzoeken moeten dan `Authorization: Bearer <token>` of `X-Opencode-Mem-Token` sturen. Open de interface met `?apiToken=<token>` zodat de browser het bewaart en meestuurt.

Dimensiemigraties genereren eerst elke nieuwe embedding, importeren die in een tijdelijke geïndexeerde shard, verifiëren het aantal rijen en vervangen pas daarna het originele bestand. Mislukte migraties laten de bron-shard ongemoeid.

## Configuratie in het kort

Configureer in `~/.config/opencode/opencode-mem.jsonc`:

**Windows:** `%USERPROFILE%\.config\opencode\opencode-mem.jsonc` (dezelfde map `.config\opencode` als hierboven — niet AppData). De standaard opslag wordt `%USERPROFILE%\.opencode-mem\data` (de `~`-vorm wordt ook op Windows uitgeklapt naar je gebruikersprofiel).

De plugin maakt bij de eerste start een volledig becommentarieerd sjabloon op dit pad. Raadpleeg [`opencode-mem.example.jsonc`](opencode-mem.example.jsonc) voor elke instelling en uitleg.

### Embeddings kiezen / configureren

Embeddings vormen de basis voor zoeken op similariteit van memories en het gebruikersprofiel. Configureer ze in hetzelfde bestand (`~/.config/opencode/opencode-mem.jsonc`). Er is **geen MLX-backend** — lokale embeddings gebruiken `@huggingface/transformers` met ONNX, niet Apple MLX.

**Lokaal (standaard):** stel alleen `embeddingModel` in. Bij het eerste gebruik wordt het model van Hugging Face gedownload en gecachet onder `{storagePath}/.cache` (standaard `~/.opencode-mem/data/.cache`).

**Remote (OpenAI-compatibel):** stel zowel `embeddingApiUrl` als `embeddingApiKey` in. De plugin roept dan `{embeddingApiUrl}/embeddings` aan met een Bearer-token. `embeddingApiKey` accepteert dezelfde geheimformaten als `memoryApiKey` (`literal`, `env://…`, `file://…`).

| Sleutel | Rol |
| --- | --- |
| `embeddingModel` | Hugging Face-id (lokaal) of API-modelnaam (remote). Standaard: `Xenova/nomic-embed-text-v1` |
| `embeddingDimensions` | Optionele overschrijving; meestal weglaten — dimensies worden opgezocht in een ingebouwde map |
| `embeddingApiUrl` | Basis-URL voor een OpenAI-compatibele embeddings-API (geen extra pad voorbij `/v1`) |
| `embeddingApiKey` | API-sleutel voor dat endpoint (vereist samen met `embeddingApiUrl`) |

Aanbevolen lokale modellen:

| Model | Dimensies | Opmerkingen |
| --- | --- | --- |
| `Xenova/nomic-embed-text-v1` | 768 | Standaard; meertalig, context van 8192 tokens |
| `Xenova/jina-embeddings-v2-base-en` | 768 | Alleen Engels, context van 8192 tokens |
| `Xenova/jina-embeddings-v2-small-en` | 512 | Sneller, context van 8192 tokens |
| `Xenova/all-MiniLM-L6-v2` | 384 | Zeer snel, context van 512 tokens |
| `Xenova/all-mpnet-base-v2` | 768 | Goede kwaliteit, context van 512 tokens |

Voorbeeld — OpenAI-embeddings op afstand:

```jsonc
{
  "embeddingApiUrl": "https://api.openai.com/v1",
  "embeddingApiKey": "env://OPENAI_API_KEY",
  "embeddingModel": "text-embedding-3-small",
}
```

Het wijzigen van `embeddingModel` (of de dimensies) kan bij de volgende start een re-embedding van opgeslagen memories veroorzaken. Kies bij voorkeur één keer een model en blijf er voor een gegeven datamap bij.

**Niet ondersteund — Intel Mac (`darwin/x64`):** lokale persistentie vereist `@tursodatabase/database`, dat geen native binding voor Intel Mac publiceert. Vaste `onnxruntime-node`-releases (`1.24.1+`, inclusief de vastgepinde `1.30.0`) missen darwin/x64 eveneens. Gebruik een Mac met Apple Silicon, Linux of Windows, of een remote endpoint via `embeddingApiUrl` + `embeddingApiKey` (zie het voorbeeld hierboven). Op ondersteunde platformen pint `opencode-mem` `onnxruntime-node@1.30.0` (fix voor het afsluiten van `Ort::Env` vanaf `1.24.1` / #225) en laadt het transformers via een CJS-resolve-shim, zodat geneste installaties van OpenCode die binding behouden. Transformers wordt naar een absoluut pad opgelost voordat die shim wordt geïnstalleerd, zodat de Bun-host van OpenCode met `--compile` niet faalt op `Cannot find module '@huggingface/transformers' from ''`. Wis na een upgrade de geneste plugincache van OpenCode (`~/.cache/opencode/packages/opencode-mem@*`) en installeer opnieuw.

### Memory-scope

- `scope: "project"`: zoekt alleen in het huidige project. Dit is de standaard.
- `scope: "all-projects"`: zoekt met `search` / `list` over alle projectshards heen.
- `memory.defaultScope` stelt de standaard queryscope in wanneer er geen expliciete scope wordt meegegeven.

### HTTP Basic Auth voor de webinterface

Als `webServerHost` op iets anders dan loopback staat (bijvoorbeeld `0.0.0.0`), is de webinterface bereikbaar voor iedereen op het netwerk. Om je memories buiten het LAN te houden, beveilig je de webserver met HTTP Basic Auth via hetzelfde configuratiebestand:

```jsonc
{
  "webServerHost": "0.0.0.0", // optioneel: interface bereikbaar maken vanaf het LAN
  "webServerAuthPassword": "kies-een-sterk-wachtwoord",
  "webServerAuthUsername": "admin", // optioneel, standaard de huidige OS-gebruiker
}
```

| Veld | Standaard | Effect |
| --- | --- | --- |
| `webServerAuthPassword` | _(leeg)_ | indien ingesteld eist de server bij elk verzoek HTTP Basic Auth-credentials. Laat leeg voor het standaardgedrag zonder authenticatie. |
| `webServerAuthUsername` | OS-gebruiker (`$USER`) | Gebruikersnaam die de Basic Auth-challenge vraagt. |

`webServerAuthPassword` accepteert dezelfde geheimformaten als `memoryApiKey`:

- een letterlijke tekenreeks (eenvoudig, prima op een persoonlijke machine),
- `env://SOME_ENV_VAR` om de waarde bij het starten uit de omgeving te halen,
- `file:///path/to/secret` om hem uit een bestand te lezen (`chmod 600` aanbevolen — de plugin waarschuwt als het bestand wereldleesbaar is).

De browser toont zijn eigen dialoogvenster voor Basic Auth en onthoudt de credentials voor de huidige sessie; als je alle browservensters sluit worden ze vergeten, dus bij heropenen moet je opnieuw inloggen. Credentials worden met een constante-tijdvergelijking gecontroleerd, en de niet-geauthenticeerde 401-respons bevat `Cache-Control: no-store`, zodat geen tussenliggende cache hem herhaalt. CORS wordt ook versochteld zodra authenticatie aanstaat, zodat andere tools op hetzelfde LAN na authenticatie met de API kunnen communiceren.

### Eén projectgeheugen delen over geneste repos

Standaard wordt een project geïdentificeerd door zijn omliggende git-repository, dus elke fysieke git-repo krijgt zijn eigen geïsoleerde geheugenopslag. Dat is verkeerd voor multi-repo-workspaces — bomen die door Google [`repo`](https://gerrit.googlesource.com/git-repo/+/HEAD/Docs/manual-repo.md) worden beheerd, monorepos of elke opstelling waarin meerdere geneste git-repositories bij één logisch project horen — omdat elke sub-repo dan afgeschermd zou zijn.

Plaats een leeg **`.opencode-mem-project`**-markerbestand in de root van de workspace:

```
my-workspace/
├── .opencode-mem-project   ← workspace-root
├── kernel/                 (eigen git-repo)
├── userspace/              (eigen git-repo)
└── tools/                  (eigen git-repo)
```

Elke sessie die ergens onder de marker wordt gestart, wordt dan aan die root gekoppeld en deelt één geheugenopslag, ongeacht in welke sub-repo de werkmap staat:

```sh
touch ~/my-workspace/.opencode-mem-project
```

De marker wordt opgezocht door vanaf de werkmap omhoog te lopen — de werkmap die elke codepad toch al meekrijgt (de werkmap van de plugin, `process.cwd()` van de web-API) — dus de identiteit is **mapgestuurd en procesonafhankelijk**. Het steunt niet op omgevingsvariabelen of een globale configuratiewaarde, wat hier onbetrouwbaar zou zijn: opencode-mem draait in meerdere opencode-processen die één webserver delen, en slechts een deel van die processen heeft een bepaalde env-var. Met de marker wordt de projectroot altijd afgeleid van waar de sessie daadwerkelijk draait.

De marker gaat vóór op git-detectie. Is hij aanwezig, dan wordt de eigen git-remote van de sub-repo bewust genegeerd (die beschrijft immers slechts één geneste repository). Zonder marker blijft het gedrag ongewijzigd (git-gebaseerde identiteit).

### Projectgeheugens verplaatsen of herstellen

opencode-mem koppelt projectshards aan een hash van de projectidentiteit. Een repository verplaatsen (OS-migratie, herstructurering van paden, wisselen van een Windows-mount naar een native pad) kan daardoor de oude shard onder `~/.opencode-mem/data/projects/` wees maken, terwijl er voor het nieuwe pad een nieuwe lege shard ontstaat.

Het gaat hieronder om aanroepen van de OpenCode-`memory`-tool met JSON-argumenten, niet om commando's voor een terminal. De issue-achtige notatie `memory migrate --from ...` komt overeen met `memory({ mode: "migrate", fromPath: "..." })`.

**1. Lokaal verplaatsen terwijl je het oude pad nog kent**

Open OpenCode in de **nieuwe** projectmap. Het doelproject mag nog geen memories bevatten (de migratie wordt bij een conflict ongewijzigd afgebroken). Bekijk eerst de gedetecteerde bron, bestemming en bestandsacties:

```typescript
memory({ mode: "migrate", fromPath: "/old/path/to/project", dryRun: true });
memory({ mode: "migrate", fromPath: "/old/path/to/project" });
```

Voor de veiligheid weigert de migratie een bron waarvan de opgeslagen projectmap nog bestaat. Wil je bewust een actieve bron verplaatsen, bekijk dan eerst de dry-run-uitvoer en geef daarna `allowLinkedSource: true` mee. De oorspronkelijke shard-bestanden van de bron worden bewaard als timestamped `*.pre-path-migrate-*.bak`-back-ups.

**2. Het oude pad is weg — ontdek eerst de wees-shard**

```typescript
memory({ mode: "list-shards" });
memory({ mode: "migrate", fromHash: "fa645294d88bbae2" });
```

`list-shards` rapporteert per projecthash de opgeslagen `projectPath`, het aantal memories en de status (`current`, `linked`, `orphaned`, `missing-file`, `empty` of `ambiguous`). `fromHash` is de 16 tekens lange lowercase hexadecimale `scopeHash` die deze aanroep teruggeeft. Gebruik die liever wanneer de oude map niet meer bestaat of meerdere shards hetzelfde opgeslagen pad bevatten, omdat git-gebaseerde identiteiten niet altijd opnieuw berekenbaar zijn vanaf een ontbrekend pad.

**3. Back-up / herstel tussen machines**

```typescript
// op de bronmachine / oude checkout
memory({ mode: "export", outputPath: "./memories.json" });

// op de doelmachine / nieuwe checkout
memory({ mode: "import", inputPath: "./memories.json", dryRun: true });
memory({ mode: "import", inputPath: "./memories.json" });
```

Export schrijft een versieerbaar JSON-document zonder vectoren. Import koppelt de memories aan het huidige project opnieuw en berekent de embeddings opnieuw met het momenteel geconfigureerde model. Import voegt memories toe aan een bestaand project, maar dubbele memory-ID's aborteren de hele import vóórdat er iets wordt geschreven; dit verschilt van `migrate`, dat een lege bestemming vereist.

Exportbestanden zijn platte tekst en kunnen memory-inhoud, gebruikersnamen en e-mailadressen, repository-URL's en absolute projectpaden bevatten. Bewaar ze zoals andere gevoelige back-ups en verwijder ze zodra ze niet meer nodig zijn. Volledig private items worden weggelaten en gebruikersprofielen en promptgeschiedenis worden niet meegenomen. Het document bevat `schemaVersion: 1`; imports wijzigen nieuwere, niet-ondersteunde schemaversies af in plaats van te gokken.

### AI-Provider voor auto-capture

Auto-capture voert een achtergrondverzoek aan de AI uit om technisch werk samen te vatten en op te slaan als memory. Het heeft één van de onderstaande providerconfiguraties nodig.

**Aanbevolen:** gebruik een provider die al in opencode geauthenticeerd is en gestructureerde output ondersteunt:

```jsonc
"opencodeProvider": "anthropic",
"opencodeModel": "claude-haiku-4-5-20251001",
```

De plugin stuurt verzoeken voor gestructureerde output via de sessie-API van opencode in plaats van rechtstreeks provider-endpoints aan te roepen, zodat opencode de authenticatie, het vernieuwen van tokens en de routering van providers beheert. De providernaam moet overeenkomen met een item uit `opencode providers list`, en het gekozen model moet gestructureerde JSON-output via opencode ondersteunen.

Ondersteunde providers: elke provider die `opencode providers list` toont (bijvoorbeeld `anthropic`, `openai`, `github-copilot`, ...).

Als `opencodeProvider` en `opencodeModel` zijn ingesteld, hebben ze voorrang op de handmatige `memoryProvider`-instellingen hieronder.

**De sessie volgen:** stel `"opencodeModel": "inherit"` in om op het moment van de aanroep een concreet OpenCode-model te gebruiken in plaats van een vastgepind id. Voor **auto-capture** wordt elke prompt vastgelegd via de `chat.params`-hook en hergebruikt het capture-verzoek de provider en het model van die prompt. Voor **profielleerwerk** en andere paden met gestructureerde output (die niet aan één gebruikersbericht zijn gebonden) valt `inherit` terug op het meest recente model in de recente-lijst van OpenCode's `model.json` (met voorkeur voor de geconfigureerde `opencodeProvider`). Het letterlijke model-id `inherit` is nooit geldig en veroorzaakte eerder `ProviderModelNotFoundError: Model not found: <provider>/inherit` op die paden. `opencodeProvider` blijft vereist als de gebruikelijke configuratiepoort.

**Terugvaloptie:** handmatige API-configuratie (als je `opencodeProvider` niet gebruikt):

```jsonc
"memoryProvider": "openai-chat",
"memoryModel": "gpt-4o-mini",
"memoryApiUrl": "https://api.openai.com/v1",
"memoryApiKey": "sk-...",
```

**Formaten voor API-sleutels:**

```jsonc
"memoryApiKey": "sk-..."
"memoryApiKey": "file://~/.config/opencode/api-key.txt"
"memoryApiKey": "env://OPENAI_API_KEY"
```

Handmatige `memoryProvider`-modi:

- `openai-chat`: OpenAI Chat Completions-compatibele API met tool/function-calling. Dit werkt met compatibele proxies zoals LiteLLM alleen als het gekozen upstream-model en de proxy tool-calls behouden.
- `openai-responses`: OpenAI Responses API met function-call-output.
- `anthropic`: Anthropic Messages API met tool use.
- `minimax`: MiniMax-endpoint dat compatibel is met de Anthropic Messages API. Stel `memoryApiUrl` in op het globale endpoint (`https://api.minimax.io`) of het Chinese endpoint (`https://api.minimaxi.com`); het pad `/anthropic/v1/messages` en de header `x-api-key` worden automatisch toegepast. Huidige modellen zijn onder meer `MiniMax-M3` (context van 1.000.000 tokens; adaptief of uitgeschakeld denken) en `MiniMax-M2.7` (context van 204.800 tokens; altijd aan denken). `MiniMax-M3` ondersteunt adaptief denken via `memoryExtraParams`.
- `orcarouter`: OpenAI-compatibele modelgateway met genamespaceerde model-ID's. `memoryApiUrl` en `memoryModel` zijn optioneel — ze vallen terug op `https://api.orcarouter.ai/v1` en `orcarouter/auto` (een routeringsalias die per verzoek een geschikt model kiest). Stel je `memoryModel` in, gebruik dan een genamespaceerd ID zoals `openai/gpt-5.5` of `deepseek/deepseek-v4-flash`; OrcaRouter weigert kale modelnamen. Voorbeeld:
  ```jsonc
  "memoryProvider": "orcarouter",
  "memoryApiKey": "<OrcaRouter API-sleutel>",
  ```
  [OrcaRouter](https://www.orcarouter.ai) voert bovendien gateway-brede zero-trust-beveiliging uit voor AI-agents op hetzelfde endpoint — elke prompt/respons wordt gecontroleerd en elke tool-call wordt standaard geweigerd tenzij expliciet toegestaan, zonder wijzigingen in applicatiecode.

Problemen oplossen:

- Fouten bij auto-capture blokkeren het handmatige gebruik van de `memory`-tool niet.
- Meldt auto-capture dat een provider niet verbonden is, controleer dan de providernaam met `opencode providers list` en configureer die provider eerst in opencode.
- Geeft een proxy of custom provider platte tekst in plaats van gestructureerde output of tool-output, kies dan een ander model of provider, of gebruik een van de handmatige providermodi hierboven.
- Voor modellen die `temperature` afwijzen: voeg `"memoryTemperature": false` toe bij handmatige API-configuratie.
- Voor modellen die geforceerde tool-calls afwijzen (`tool_choice: "required"`, bijvoorbeeld sommige denkmodi): voeg `"forceToolChoice": false` toe bij `openai-chat` / `orcarouter`.
- **Niet-ondersteunde platformen:** Intel Mac (`darwin/x64`) wordt niet ondersteund — `@tursodatabase/database` en de vaste `onnxruntime-node`-releases (vastgepind op `1.30.0`) bevatten geen x64-native binding. Gebruik Apple Silicon, Linux of Windows, of een remote embedding-endpoint via `embeddingApiUrl` + `embeddingApiKey`. MLX wordt niet ondersteund.

## Publieke subpad-exports

Naast het-hoofdentrypoint van de plugin stelt `opencode-mem` één stabiel subpad beschikbaar dat andere opencode-plugins direct kunnen importeren. Zo hoef je bij het schrijven van tools van derden die dezelfde memoryopslag lezen of beschrijven niet de conventies voor containertags te reverse-engineeren.

### `opencode-mem/tags`

Canonieke helpers voor containertags. Dezelfde functies die opencode-mem zelf gebruikt om auto-captured memories te scopen.

```ts
import { getProjectTagInfo, getUserTagInfo, getTags } from "opencode-mem/tags";

// Canonieke projecttag afgeleid van cwd (git-remote-URL indien aanwezig,
// anders het pad van de projectroot). Formaat: `opencode_project_<sha16>`.
const projectTag = getProjectTagInfo(process.cwd()).tag;

// Canonieke gebruikertag afgeleid van `git config user.email`.
// Formaat: `opencode_user_<sha16>`.
const userTag = getUserTagInfo().tag;

// Beide tegelijk.
const { user, project } = getTags(process.cwd());
```

Tags die deze helpers produceren komen overeen met wat auto-capture schrijft, dus plugins van derden die `POST /api/memories` aanroepen belanden in dezelfde shards die de rest van het systeem al begrijpt. Zelfgebouwde tags waarvan het fragment niet `_project_` of `_user_` is, belanden in schaduwshards die `/api/stats` en `/api/memories` stilzwijgend wegfilteren — met deze helpers vermijd je die valkuil.

## Ontwikkeling en bijdragen

Lokaal bouwen en testen:

```bash
bun install
bun run build
bun run typecheck
bun run format
```

Dit project zoekt actief bijdragen om het definitieve memory-plugin voor AI-codeeragenten te worden. Of je nu bugs verhelpt, functies toevoegt, documentatie verbetert of de ondersteuning voor embeddingmodellen uitbreidt — je bijdragen zijn essentieel. De codebase is goed gestructureerd en klaar voor uitbreiding. Gebruik de Issue- of Feature-request-sjablonen en vul het pull-request-sjabloon in wanneer je een PR indient — we reviewen en mergen bijdragen snel.

## Licentie en links

MIT-licentie — zie het bestand LICENSE

- **Repository**: https://github.com/tickernelz/opencode-mem
- **Issues**: https://github.com/tickernelz/opencode-mem/issues
- **OpenCode-platform**: https://opencode.ai

Geïnspireerd door [opencode-supermemory](https://github.com/supermemoryai/opencode-supermemory)
