# PostgreSQL nella Case Platform

## Scopo

PostgreSQL è il DBMS relazionale standard della Case Platform.

La piattaforma lo utilizza sia per le componenti infrastrutturali che richiedono persistenza relazionale, come Keycloak, sia per i dati applicativi appartenenti ai singoli domini di business.

Questo documento descrive il ruolo di PostgreSQL nell'architettura, il modello di separazione dei dati e le responsabilità delle componenti. Le procedure di installazione e gestione operativa sono trattate nella documentazione dedicata agli ambienti.

## Principi architetturali

L'utilizzo di PostgreSQL segue questi principi:

- ogni dominio è proprietario dei propri dati;
- ogni componente accede esclusivamente al database di propria competenza;
- i dati di autenticazione e autorizzazione sono separati dai dati applicativi;
- credenziali e permessi sono distinti per database;
- le modifiche allo schema sono versionate e gestite dalla componente proprietaria;
- non sono consentiti accessi diretti al database di un altro dominio;
- le integrazioni tra domini avvengono tramite API o eventi;
- la separazione logica locale deve consentire una futura separazione fisica senza modificare il modello applicativo.

Utilizziamo CloudNativePG come operatore che automatizza il ciclo di vita di PostgreSQL su Kubernetes.

## Organizzazione dei database

Nell'ambiente locale può essere utilizzato un unico cluster PostgreSQL, all'interno del quale vengono creati database separati:

```text
Cluster PostgreSQL
├── database keycloak
│   └── utente keycloak
├── database anc
│   └── utente anc
├── database successioni
│   └── utente successioni
└── database <dominio>
    └── utente <dominio>
```

Il cluster PostgreSQL costituisce l'unità infrastrutturale condivisa. I database rappresentano invece confini logici indipendenti, associati alle rispettive componenti o ai rispettivi domini.

In ambienti più articolati, i database potranno essere distribuiti su cluster PostgreSQL distinti per esigenze di disponibilità, sicurezza, carico, manutenzione o vincoli del cliente.

## Database delle componenti di piattaforma

Le componenti infrastrutturali che richiedono persistenza dispongono di un database dedicato.

### Keycloak

Keycloak utilizza il database `keycloak` per conservare, tra gli altri:

- realm;
- client OIDC e SAML;
- utenti eventualmente gestiti localmente;
- gruppi e ruoli;
- associazioni tra utenti, gruppi e ruoli;
- configurazioni degli Identity Provider;
- sessioni persistenti;
- configurazioni ed eventi amministrativi.

Keycloak accede al database mediante un'utenza dedicata. Nessun servizio applicativo deve leggere o modificare direttamente le tabelle di Keycloak.

Le informazioni di identità e autorizzazione necessarie alle applicazioni sono ottenute attraverso i protocolli e le API esposte da Keycloak, non interrogando il suo database.

## Database dei domini

Ogni dominio di business dispone di un proprio database.

Per esempio, il dominio ANC utilizza il database `anc` per i dati relativi alle proprie pratiche, lavorazioni e regole di persistenza. Il dominio Successioni utilizza un database distinto e non accede direttamente alle tabelle ANC.

Questa separazione garantisce:

- autonomia evolutiva;
- ownership chiara dei dati;
- isolamento degli accessi;
- riduzione dell'accoppiamento;
- migrazioni indipendenti;
- possibilità di distribuire separatamente i domini.

La presenza iniziale di più moduli o servizi nello stesso repository di dominio non modifica il principio di ownership: il database continua ad appartenere al dominio e non alla piattaforma nel suo complesso.

## Accesso e sicurezza

Per ogni database devono essere previsti:

- un proprietario dedicato;
- credenziali specifiche;
- privilegi limitati alle operazioni necessarie;
- connessioni ammesse soltanto dalle componenti autorizzate;
- segreti non presenti in chiaro nei repository Git;
- cifratura delle connessioni negli ambienti che la richiedono;
- tracciamento e rotazione delle credenziali secondo le politiche dell'ambiente.

Nell'ambiente Kubernetes le credenziali vengono fornite alle applicazioni tramite riferimenti a Secret. I manifest versionati non devono contenere password o altri dati sensibili.

## Gestione dello schema

Ogni componente proprietaria del database gestisce autonomamente l'evoluzione del proprio schema.

Le migrazioni devono essere:

- versionate insieme al codice della componente;
- ripetibili negli ambienti previsti;
- eseguite in modo controllato durante il rilascio;
- compatibili con le strategie di rollback e aggiornamento;
- verificate automaticamente nelle pipeline.

Per i database applicativi verrà adottato uno strumento di migrazione compatibile con lo stack Quarkus e con PostgreSQL. La scelta e la configurazione dello strumento saranno definite nella baseline di sviluppo backend.

Lo schema interno di Keycloak è invece gestito da Keycloak stesso e non deve essere modificato mediante migrazioni applicative.

## Persistenza in Kubernetes

Nell'ambiente locale PostgreSQL viene eseguito nel cluster k3s e utilizza storage persistente.

La distribuzione deve prevedere almeno:

- workload stateful;
- PersistentVolumeClaim;
- Service interno al cluster;
- configurazione esterna all'immagine;
- Secret per le credenziali;
- probe di disponibilità;
- limiti e richieste di risorse;
- versione PostgreSQL fissata esplicitamente;
- gestione dichiarativa tramite GitOps.

Il database non deve essere esposto pubblicamente. Gli strumenti amministrativi possono collegarsi attraverso modalità controllate, ad esempio port-forward temporaneo.

## Backup e ripristino

La persistenza su volume non sostituisce il backup.

La strategia operativa deve considerare:

- backup periodici;
- conservazione dei backup;
- cifratura;
- verifica dell'integrità;
- prove di ripristino;
- Recovery Point Objective e Recovery Time Objective;
- trattamento distinto dei database in base alla loro criticità.

Le modalità concrete possono variare tra ambiente locale e installazioni presso i clienti, senza modificare il modello logico descritto in questo documento.

## Portabilità della piattaforma

La piattaforma non assume che tutti i database debbano risiedere nello stesso cluster PostgreSQL.

L'ambiente locale adotta inizialmente un cluster condiviso per contenere il consumo di risorse e semplificare la gestione. La separazione per database, utenze, configurazioni e migrazioni consente successivamente di adottare:

- un cluster dedicato alle componenti di piattaforma;
- cluster separati per domini critici;
- servizi PostgreSQL gestiti dal cliente o da un cloud provider;
- differenti politiche di disponibilità e backup.

Questa organizzazione permette di adattare la Case Platform ai diversi contesti infrastrutturali mantenendo invariati i confini applicativi e la proprietà dei dati.

## Responsabilità

| Ambito | Responsabilità |
| --- | --- |
| Piattaforma | Provisioning PostgreSQL, storage, sicurezza, monitoraggio, backup e standard comuni |
| Keycloak | Gestione e aggiornamento del proprio schema |
| Dominio | Modello dati, migrazioni e accesso al proprio database |
| GitOps | Distribuzione dichiarativa delle risorse e delle configurazioni non sensibili |
| Pipeline CI/CD | Validazione delle migrazioni e compatibilità tra codice e schema |

## Documentazione correlata

Le istruzioni per installare PostgreSQL nell'ambiente locale k3s, configurare la persistenza e verificarne il funzionamento saranno descritte in:

```text
development/local-environment/postgresql-installation.md
```

Le convenzioni per le migrazioni dei database applicativi saranno descritte nella documentazione di sviluppo backend.
