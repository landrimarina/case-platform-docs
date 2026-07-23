# Case Platform — Documentazione

## Scopo

Il repository `case-platform-docs` contiene la documentazione trasversale della Case Platform:

- architettura complessiva;
- decisioni architetturali;
- standard tecnici;
- baseline tecnologica;
- istruzioni per lo sviluppo;
- linee guida per gli agenti AI;
- diagrammi e glossario condiviso.

La documentazione specifica di una singola applicazione o dominio risiede invece nella directory `docs` del relativo repository full-stack, per esempio:

```text
case-platform-anc/docs
case-platform-successioni/docs
```

## Struttura del repository

```text
case-platform-docs/
├── architecture/
│   ├── frontend/
│   ├── backend/
│   └── platform/
├── adr/
│   ├── frontend/
│   ├── backend/
│   └── platform/
├── standards/
│   ├── frontend/
│   ├── backend/
│   ├── api/
│   └── security/
├── development/
├── ai-guidelines/
├── diagrams/
├── glossary/
├── .github/
│   └── workflows/
├── AGENTS.md
├── baselineTecnologica.md
└── README.md
```

## Directory

### `architecture`

Contiene la documentazione dell’architettura complessiva della Case Platform, organizzata per area tecnologica.

#### `architecture/frontend`

Descrive l’architettura frontend condivisa:

- Application Shell;
- microfrontend;
- composizione e caricamento dei MFE;
- librerie UI e Frontend Core;
- Module Federation;
- Platform Event Bus;
- gestione della sessione frontend;
- contratti e SDK TypeScript;
- dipendenze consentite tra i componenti.

La documentazione funzionale e tecnica di un MFE specifico rimane nel repository del relativo dominio.

#### `architecture/backend`

Descrive l’architettura backend condivisa:

- BFF;
- Shared Platform Capabilities;
- Application Services;
- Business Domain;
- separazione tra Application, Domain e Infrastructure;
- integrazioni;
- persistenza;
- sicurezza dei servizi;
- comunicazioni sincrone e asincrone.

Le regole di business specifiche rimangono nei repository dei domini.

#### `architecture/platform`

Descrive le capability infrastrutturali e operative:

- Kubernetes e k3s;
- GitOps e Argo CD;
- IAM e Keycloak;
- API Gateway;
- observability;
- logging, metriche e tracing;
- configurazione degli ambienti;
- gestione dei secret;
- deployment e scalabilità.

### `adr`

Contiene gli Architecture Decision Record, cioè le decisioni architetturali rilevanti con:

- contesto;
- problema;
- alternative considerate;
- decisione;
- motivazioni;
- conseguenze.

Ogni ADR deve essere numerato. Una decisione superata non viene cancellata: un nuovo ADR la sostituisce indicandone esplicitamente lo stato.

#### `adr/frontend`

Contiene le decisioni relative al frontend, per esempio:

- adozione dei microfrontend;
- utilizzo di Vite e Module Federation;
- responsabilità della Shell;
- utilizzo dell’Event Bus;
- strategia delle librerie condivise;
- gestione dello stato frontend.

#### `adr/backend`

Contiene le decisioni relative al backend, per esempio:

- adozione del runtime applicativo;
- organizzazione dei layer;
- separazione dei domini;
- utilizzo del BFF;
- strategia delle transazioni;
- integrazione con i sistemi esterni.

#### `adr/platform`

Contiene le decisioni relative alla piattaforma, per esempio:

- utilizzo di Kubernetes o k3s;
- adozione di Argo CD;
- organizzazione dei repository;
- strategia Git;
- utilizzo di Keycloak;
- gestione degli ambienti;
- tecnologie di observability.

### `standards`

Contiene le regole obbligatorie e le convenzioni comuni a tutti i repository.

Gli ADR spiegano perché è stata presa una decisione; gli standard stabiliscono come deve essere applicata.

#### `standards/frontend`

Contiene gli standard di sviluppo frontend:

- struttura dei progetti React;
- convenzioni TypeScript;
- naming di componenti, hook e file;
- utilizzo delle librerie condivise;
- accessibilità;
- gestione degli errori;
- testing;
- integrazione dei MFE;
- regole per gli eventi frontend.

#### `standards/backend`

Contiene gli standard di sviluppo backend:

- struttura dei package Java;
- convenzioni di naming;
- separazione tra layer;
- gestione delle eccezioni;
- logging;
- transazioni;
- configurazione;
- test unitari e di integrazione;
- dipendenze consentite.

#### `standards/api`

Contiene gli standard per progettazione e versionamento delle API:

- convenzioni REST;
- naming degli endpoint;
- metodi HTTP;
- codici di risposta;
- modello uniforme degli errori;
- paginazione e filtri;
- versionamento;
- correlation ID;
- regole OpenAPI;
- compatibilità e breaking change.

I contratti specifici dei domini risiedono nei relativi repository full-stack.

#### `standards/security`

Contiene gli standard di sicurezza comuni:

- autenticazione e autorizzazione;
- OAuth 2.0 e OpenID Connect;
- gestione dei ruoli;
- mapping tra gruppi aziendali e ruoli applicativi;
- protezione delle API;
- gestione di token, cookie e sessioni;
- gestione dei secret;
- TLS e mTLS;
- least privilege;
- audit di sicurezza.

### `development`

Contiene le istruzioni operative comuni per sviluppatori e agenti AI:

- preparazione dell’ambiente locale;
- installazione degli strumenti;
- utilizzo di Git;
- strategia dei branch;
- build e test;
- avvio locale dei componenti;
- utilizzo di k3s;
- troubleshooting;
- modalità di rilascio.

Non contiene documentazione funzionale specifica dei domini.

### `ai-guidelines`

Contiene le istruzioni centrali per gli agenti AI:

- confini architetturali;
- responsabilità dei repository;
- file modificabili;
- codice generato che non deve essere modificato;
- controlli obbligatori;
- regole per test e documentazione;
- Definition of Done;
- criteri di code review;
- gestione delle dipendenze;
- operazioni Git consentite;
- regole di sicurezza.

I file `AGENTS.md` presenti nei singoli repository devono essere coerenti con queste linee guida e possono aggiungere regole specifiche del componente o del dominio.

### `diagrams`

Contiene i diagrammi architetturali e i relativi sorgenti modificabili:

- viste logiche;
- viste applicative;
- viste tecnologiche;
- flussi di autenticazione;
- flussi di deployment;
- dipendenze tra componenti;
- sequence diagram;
- diagrammi di integrazione.

Quando possibile, devono essere conservati sia il sorgente sia il formato esportato.

### `glossary`

Contiene il glossario comune:

- termini architetturali;
- acronimi;
- componenti della piattaforma;
- terminologia condivisa tra frontend e backend;
- definizioni utilizzate nei contratti e nella documentazione.

I termini strettamente funzionali di un dominio devono essere documentati nel glossario del relativo repository.

### `.github/workflows`

Contiene le pipeline GitHub Actions relative alla documentazione:

- validazione Markdown;
- verifica dei collegamenti;
- controllo della struttura;
- eventuale generazione del sito documentale;
- controlli automatici su ADR e standard.

## File principali

### `AGENTS.md`

Contiene le istruzioni che gli agenti AI devono leggere prima di modificare il repository:

- ambito;
- convenzioni;
- controlli da eseguire;
- modalità di aggiornamento;
- operazioni vietate.

### `baselineTecnologica.md`

Descrive la baseline tecnologica della Case Platform:

- tecnologie approvate;
- versioni di riferimento;
- runtime;
- framework;
- database;
- componenti infrastrutturali;
- strumenti DevOps;
- criteri di aggiornamento.

Le versioni devono essere esplicite e, quando necessario, accompagnate dalla motivazione della scelta.

### `README.md`

Presenta lo scopo del repository, la struttura, le responsabilità delle directory e le regole generali di utilizzo.

## Documentazione centrale e documentazione di dominio

| Contenuto | Collocazione |
| --- | --- |
| Architettura trasversale | `case-platform-docs/architecture` |
| Standard comuni | `case-platform-docs/standards` |
| Decisioni architetturali comuni | `case-platform-docs/adr` |
| Linee guida per gli agenti AI | `case-platform-docs/ai-guidelines` |
| Funzionalità ANC | `case-platform-anc/docs/functional` |
| Regole di business ANC | `case-platform-anc/docs/functional` |
| Architettura specifica ANC | `case-platform-anc/docs/architecture` |
| Decisioni locali ANC | `case-platform-anc/docs/adr` |
| Contratto OpenAPI ANC | `case-platform-anc/contracts` |

La documentazione centrale stabilisce principi e regole comuni; la documentazione di dominio descrive la loro applicazione alla singola applicazione.

## Convenzioni documentali

- Utilizzare Markdown.
- Usare nomi di file descrittivi in `kebab-case`.
- Inserire titolo, scopo e stato del documento.
- Collegare i documenti correlati con percorsi relativi.
- Evitare duplicazioni tra documentazione centrale e documentazione di dominio.
- Aggiornare la documentazione nello stesso change set della modifica tecnica.
- Non inserire password, token, secret o altre informazioni riservate.

## Flusso di aggiornamento

Le modifiche seguono la strategia Git della piattaforma:

```text
feature/* → develop → main
```

1. creare un branch `feature/*` da `develop`;
2. aggiornare o aggiungere la documentazione;
3. eseguire le verifiche previste;
4. aprire una pull request verso `develop`;
5. promuovere su `main` dopo approvazione e consolidamento.


