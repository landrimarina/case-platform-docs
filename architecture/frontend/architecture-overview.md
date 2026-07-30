# Case Platform — Architettura Frontend

## 1. Scopo del documento

Questo documento descrive l'architettura frontend della Case Platform e costituisce un riferimento condiviso per progettazione, sviluppo e attività svolte tramite agenti AI.

Il perimetro comprende:

- Application Shell;
- microfrontend di dominio e trasversali;
- librerie frontend condivise;
- contratti OpenAPI e SDK TypeScript;
- comunicazione tra microfrontend tramite Event Bus;
- configurazione, versionamento, rilascio e regole di dipendenza.

Il BFF e l'architettura backend non rientrano nel perimetro di questo documento.

L'obiettivo è impostare il vertical slice con caratteristiche architetturali e operative coerenti con una piattaforma destinata allo sviluppo reale.

---

## 2. Principi architetturali

1. La Shell governa la composizione dell'esperienza utente e carica i microfrontend.
2. Ogni microfrontend è un'applicazione autonoma, compilabile e rilasciabile indipendentemente.
3. Un microfrontend non importa direttamente il sorgente di un altro microfrontend.
4. Il codice riutilizzabile viene distribuito attraverso librerie npm versionate.
5. I contratti delle API sono centralizzati e costituiscono la fonte ufficiale per generare gli SDK TypeScript.
6. Gli endpoint non vengono scritti direttamente nelle schermate React.
7. Le comunicazioni runtime disaccoppiate tra Shell e microfrontend utilizzano eventi tipizzati.
8. L'Event Bus non sostituisce le API, lo stato applicativo o la persistenza.
9. Le dipendenze devono essere unidirezionali e prive di cicli.
10. Configurazioni e versioni devono essere esplicite e riproducibili per consentire uno sviluppo affidabile tramite AI.

---

## 3. Repository frontend

| Repository                      | Responsabilità                                                                          | Modalità di distribuzione         |
| --------------------------------| --------------------------------------------------------------------------------------- | ----------------------------------|
| `case-platform-shell`           | Layout, navigazione, routing, caricamento dei MFE, sessione e composizione della pagina | Applicazione indipendente         |
| `case-platform-mfe-anc`         | Funzionalità frontend specifiche del dominio ANC                                        | Applicazione indipendente         |
| `case-platform-mfe-successioni` | Funzionalità frontend specifiche del dominio Successioni                                | Applicazione indipendente         |
| `case-platform-mfe-common`      | Funzionalità trasversali autonome, come profilo, centro notifiche e ricerca globale     | Applicazione indipendente         |
| `case-platform-ui`              | Design System, tema Poste, accessibilità e componenti React condivisi                   | Libreria npm versionata           |
| `case-platform-frontend-core`   | Event Bus, logging, gestione tecnica degli errori, configurazione e utility             | Libreria npm versionata           |
| `case-platform-contracts`       | Contratti OpenAPI, contratti degli eventi frontend e configurazioni di generazione      | Sorgente di artefatti versionati  |
| `case-platform-gitops`          | Configurazione degli ambienti e versioni degli artefatti distribuiti                    | Repository GitOps                 |
| `case-platform-docs`            | Architettura, ADR, standard, convenzioni e istruzioni per gli agenti AI                 | Documentazione versionata         |

---

## 4. Microfrontend e librerie: differenza fondamentale

Un microfrontend ha un proprio ciclo di build e rilascio e viene caricata dalla Shell a runtime.
Una libreria viene invece importata nel sorgente, risolta durante la build e utilizzata come dipendenza versionata.

| Elemento          | Come viene integrato              | Come viene rilasciato               |
| ------------------| --------------------------------- | ------------------------------------|
| Microfrontend     | Caricato dalla Shell a runtime    | Indipendentemente                   |
| Libreria UI       | Import npm nel sorgente           | Pacchetto npm versionato            |
| Frontend Core     | Import npm nel sorgente           | Pacchetto npm versionato            |
| SDK API           | Import npm nel sorgente           | Pacchetto npm generato e versionato |
| Contratti eventi  | Import npm dei tipi               | Pacchetto npm generato e versionato |

Regola pratica:

- se è una funzionalità autonoma che occupa una regione della pagina, può essere un MFE;
- se deve essere utilizzata nel codice di più MFE, è una libreria.

---

## 5. Application Shell

La Shell costituisce il contenitore dell'esperienza frontend. Non contiene le funzionalità di business dei domini.

Responsabilità principali:

- layout generale;
- menu e navigazione;
- routing principale;
- caricamento dinamico dei microfrontend;
- gestione del ciclo di montaggio e smontaggio dei MFE;
- gestione della sessione lato frontend;
- propagazione del contesto iniziale;
- gestione del tema e della lingua;
- gestione degli errori di caricamento dei MFE;
- ascolto degli eventi globali di propria competenza.

Esempio di composizione:

```text
Application Shell
├── MFE Common       profilo e notifiche
└── MFE ANC          area funzionale ANC
```

Quando l'utente apre una rotta come `/anc/pratiche`, la Shell:

1. riconosce la rotta;
2. individua il MFE ANC;
3. carica il relativo artefatto remoto;
4. recupera il modulo esposto;
5. monta il componente nella regione prevista;
6. passa il contesto iniziale definito dal contratto di montaggio.

Con Vite o Module Federation, ogni MFE produce un artefatto remoto, normalmente esposto attraverso un file come `remoteEntry.js`. Gli indirizzi degli artefatti devono essere configurabili per ambiente e non codificati rigidamente nella Shell.

---

## 6. Microfrontend di dominio

I MFE di dominio contengono esclusivamente funzionalità appartenenti al relativo dominio.

Esempi:

- `case-platform-mfe-anc`: ricerca, lavorazione e visualizzazione delle pratiche ANC;
- `case-platform-mfe-successioni`: gestione delle pratiche di Successioni.

Ogni MFE di dominio può dipendere da:

@case-platform identifica il namespace logico delle librerie, mentre il nome tecnico definitivo dipenderà dal registry scelto.

```text
@case-platform/ui
@case-platform/frontend-core
@case-platform/{dominio}-api-client
@case-platform/frontend-events
```
Repository(es. anc):

case-platform-frontend/
└── libs/
    ├── ui
    └── frontend-core

case-platform-contracts/
└── frontend-events

case-platform-anc/
└── contracts/
    └── openapi/
        └── anc-api.yaml

Da questi sorgenti vengono prodotti i pacchetti:
@case-platform/ui
@case-platform/frontend-core
@case-platform/{dominio}-api-client
@case-platform/frontend-events

Un MFE di dominio non può:

- importare un altro MFE;
- contenere copie locali dei modelli generati da OpenAPI;
- modificare il codice generato degli SDK;
- incorporare componenti grafici condivisi duplicando il Design System;
- conoscere l'indirizzo fisico dei servizi;
- utilizzare l'Event Bus per trasferire o persistere dati di business.

---

## 7. Microfrontend comuni

`case-platform-mfe-common` contiene funzionalità trasversali complete e autonome, per esempio:

- profilo operatore;
- centro notifiche;
- ricerca globale;
- selezione del contesto operativo.

La Shell carica e posiziona il MFE Common. Gli altri MFE non ne importano il codice.

Se ANC deve chiedere la visualizzazione di una notifica, pubblica un evento. Il MFE Common, se caricato e registrato, riceve l'evento e visualizza la notifica.

Se una funzionalità deve essere incorporata direttamente nel codice di ANC, non deve essere collocata nel MFE Common. Deve essere valutata come componente importabile di `case-platform-ui` o servizio di `case-platform-frontend-core`.

---

## 8. Libreria UI

`case-platform-ui` contiene il Design System e i componenti visuali riutilizzabili:

- tema Poste;
- colori, tipografia, spaziature e icone;
- pulsanti e campi;
- form e validazioni visuali;
- tabelle;
- modali e dialoghi di conferma;
- indicatori di caricamento;
- componenti accessibili;
- pattern di layout condivisi.

Esempio di utilizzo:

```typescript
import {
  Button,
  TextField,
  ConfirmDialog
} from "@case-platform/ui";
```

La libreria UI:

- non dipende dai MFE;
- non contiene logica di dominio;
- non effettua direttamente chiamate alle API di dominio;
- viene pubblicata con versionamento semantico.

---

## 9. Frontend Core

`case-platform-frontend-core` contiene servizi e utility tecniche importabili senza interfaccia grafica:

- implementazione del `PlatformEventBus`;
- logging frontend;
- correlation ID e contesto tecnico;
- gestione uniforme degli errori tecnici;
- caricamento della configurazione runtime;
- funzioni e tipi tecnici comuni.

Esempio di utilizzo:

```typescript
import {
  eventBus,
  logger
} from "@case-platform/frontend-core";
```

## 10. Contratti OpenAPI e SDK TypeScript

`case-platform-contracts` è la fonte ufficiale e unica dei contratti API.

Struttura indicativa:

```text
case-platform-contracts/
├── openapi/
│   ├── platform-api.yaml
│   ├── anc-api.yaml
│   └── successioni-api.yaml
├── frontend-events/
├── generator-config/
│   ├── typescript.yaml
│   └── java.yaml
├── rules/
├── tests/
└── .github/workflows/
```

Esempio di operazione dichiarata nel contratto Successioni:

```text
POST /api/successioni/pratiche
operationId: savePratica
request: PraticaRequest
response: PraticaResponse
```

La pipeline dei contratti deve:

1. validare la sintassi OpenAPI;
2. applicare le regole architetturali;
3. verificare eventuali modifiche incompatibili;
4. generare gli SDK TypeScript;
5. eseguire i test;
6. assegnare una versione;
7. pubblicare i pacchetti nel registry npm.

Esempi di pacchetti prodotti:

```text
@case-platform/anc-api-client:1.0.0
@case-platform/successioni-api-client:1.0.0
@case-platform/platform-api-client:1.0.0
@case-platform/frontend-events:1.0.0
```

Il codice generato non deve essere copiato o modificato manualmente nei MFE.

---

## 11. Wrapper API nei microfrontend

Ogni MFE utilizza l'SDK TypeScript, una libreria generata dal contratto OpenAPI.

Struttura indicativa:

```text
case-platform-mfe-successioni/
└── src/
    └── api/
        └── successioniClient.ts
```

Esempio:

```typescript
import { SuccessioniApi }
  from "@case-platform/successioni-api-client";

const api = new SuccessioniApi();

export const successioniClient = {
  savePratica: api.savePratica.bind(api)
};
```

La schermata React richiama il wrapper:

```typescript
await successioniClient.savePratica(formData);
```

Il wrapper consente di:

- isolare la schermata dall'SDK generato;
- applicare comportamenti frontend specifici;
- semplificare i test;
- evitare endpoint sparsi nei componenti React;
- sostituire o aggiornare l'SDK senza propagare modifiche in tutta l'interfaccia.

Il wrapper non deve duplicare i modelli del contratto né introdurre regole di business.

---

## 12. Platform Event Bus

### 12.1 Definizione

Il `PlatformEventBus` è un servizio TypeScript eseguito nel browser. Appartiene a `case-platform-frontend-core` e consente comunicazioni runtime disaccoppiate tra Shell e microfrontend già caricati.

Implementazione prevista:

```text
EventTarget
CustomEvent
window.addEventListener()
window.dispatchEvent()
```

Shell e MFE condividono lo stesso oggetto `window`. `frontend-core` espone un'interfaccia controllata e impedisce l'uso diretto e non governato delle API browser.

Struttura indicativa:

```text
case-platform-frontend-core/
└── src/
    └── events/
        ├── PlatformEventBus.ts
        └── index.ts
```

### 12.2 Quando serve

L'Event Bus serve quando un MFE deve segnalare un cambiamento alla Shell o a un altro MFE senza conoscerne l'implementazione e senza importarlo.

Casi appropriati:

- richiesta di una notifica globale;
- richiesta di navigazione alla Shell;
- segnalazione della scadenza della sessione;
- cambiamento del contesto operativo;
- sincronizzazione di informazioni temporanee tra componenti autonomi già attivi.

Esempio: il MFE Common modifica il contesto dell'operatore.

```text
MFE Common
    ↓ pubblica platform.operator-context.changed
Platform Event Bus
    ↓ distribuisce l'evento
MFE ANC
    ↓ aggiorna i dati visualizzati
```

Esempio di pubblicazione:

```typescript
eventBus.publish("platform.operator-context.changed", {
  officeId: "NAPOLI-01",
  role: "OPERATORE"
});
```

Esempio di sottoscrizione:

```typescript
const unsubscribe = eventBus.subscribe(
  "platform.operator-context.changed",
  context => reloadPractices(context.officeId)
);
```

Quando il componente viene smontato deve rimuovere la sottoscrizione:

```typescript
unsubscribe();
```

### 12.3 Quando non serve

L'Event Bus non deve essere utilizzato:

- tra componenti appartenenti allo stesso MFE: usare props, callback o stato React;
- per chiamare servizi remoti: usare gli SDK API;
- per persistere dati;
- per trasferire lo stato ufficiale di una pratica;
- per sostituire lo state management;
- quando è richiesta consegna garantita;
- per comunicazioni differite;
- per nascondere una dipendenza funzionale diretta non progettata.

Gli eventi sono temporanei: se il destinatario non è caricato o non è ancora registrato, l'evento può non essere ricevuto.

### 12.4 Dati iniziali ed eventi successivi

La Shell passa al MFE le informazioni indispensabili alla sua inizializzazione attraverso un contratto di montaggio, per esempio:

- lingua;
- tema;
- identificativo e contesto dell'utente;
- configurazione runtime consentita.

L'Event Bus comunica invece cambiamenti avvenuti dopo il caricamento:

```text
Dati iniziali necessari al MFE
→ contratto di montaggio gestito dalla Shell

Cambiamento durante l'utilizzo
→ Platform Event Bus
```

### 12.5 Contratti tipizzati degli eventi

I nomi e i payload degli eventi non devono essere stringhe inventate liberamente nei MFE. Devono essere definiti centralmente e pubblicati come pacchetto versionato:

```text
@case-platform/frontend-events
```

Esempio:

```typescript
export interface PlatformEventMap {
  "platform.notification.requested": {
    type: "success" | "warning" | "error";
    message: string;
  };

  "platform.navigation.requested": {
    path: string;
  };

  "platform.session.expired": {
    reason: string;
  };

  "platform.operator-context.changed": {
    officeId: string;
    role: string;
  };
}
```

`frontend-core` implementa il meccanismo publish/subscribe e utilizza questa mappa per garantire la correttezza TypeScript.

---

## 13. Compatibilità browser

`EventTarget`, `CustomEvent`, `addEventListener` e `dispatchEvent` sono API standard del DOM supportate dai browser moderni.

La piattaforma deve comunque definire una matrice ufficiale, per esempio:

```text
Chrome: ultime 2 versioni
Edge: ultime 2 versioni
Firefox: ultime 2 versioni
```

La pipeline frontend deve eseguire test automatici sui browser supportati. La compatibilità non deve essere affidata soltanto all'esperienza dello sviluppatore.

---

## 14. Dipendenze consentite

```text
Application Shell
├── carica MFE ANC
├── carica MFE Successioni
└── carica MFE Common

MFE ANC
├── importa UI
├── importa Frontend Core
├── importa ANC API Client
└── importa Frontend Events

MFE Successioni
├── importa UI
├── importa Frontend Core
├── importa Successioni API Client
└── importa Frontend Events

MFE Common
├── importa UI
├── importa Frontend Core
├── importa Platform API Client
└── importa Frontend Events
```

Matrice sintetica:

| Sorgente        | Può dipendere da                                              | Non può dipendere da                    |
| --------------- | ------------------------------------------------------------- | --------------------------------------- |
| Shell           | UI, Frontend Core, contratti, MFE a runtime                   | Logica interna dei domini               |
| MFE di dominio  | UI, Frontend Core, proprio SDK, eventi                        | Altri MFE, SDK di domini non necessari  |
| MFE Common      | UI, Frontend Core, Platform SDK, eventi                       | MFE di dominio                          |
| UI              | dipendenze grafiche approvate                                 | MFE, domini, API di dominio             |
| Frontend Core   | contratti tecnici ed eventi                                   | UI, MFE, domini                         |
| SDK generati    | runtime tecnico strettamente necessario                       | MFE e logica applicativa                |

Sono vietate le dipendenze circolari.

---

## 15. Flussi di esempio

### 15.1 Apertura di ANC

```text
Utente apre /anc/pratiche
→ Shell riconosce la rotta
→ Shell carica MFE ANC
→ Shell passa il contesto iniziale
→ MFE ANC importa UI, Frontend Core e ANC API Client
→ MFE ANC visualizza la funzionalità
```

### 15.2 Salvataggio e notifica

```text
Operatore seleziona Save nel MFE ANC
→ schermata richiama ancClient
→ ancClient utilizza ANC API Client generato
→ operazione completata
→ MFE ANC pubblica platform.notification.requested
→ MFE Common riceve l'evento
→ MFE Common mostra la notifica
```

### 15.3 Cambio del contesto operatore

```text
Operatore cambia ufficio nel MFE Common
→ MFE Common pubblica platform.operator-context.changed
→ MFE ANC riceve l'evento
→ MFE ANC aggiorna la propria vista
```

---

## 16. Versionamento e rilascio

Ogni componente possiede un ciclo di rilascio indipendente.

Applicazioni:

- Shell e MFE producono artefatti frontend distribuibili;
- ogni rilascio possiede una versione identificabile;
- la compatibilità tra Shell e MFE deve essere verificata automaticamente.

Librerie:

- UI, Frontend Core, SDK ed eventi sono pacchetti npm;
- le versioni devono essere dichiarate esplicitamente nei `package.json`;
- deve essere adottato il versionamento semantico;
- una modifica incompatibile richiede una major version;
- la pipeline deve bloccare modifiche incompatibili non dichiarate.

La configurazione GitOps determina quali versioni vengono distribuite in ogni ambiente.

---

## 17. Regole operative per gli agenti AI

Un agente AI che modifica un repository frontend deve:

1. leggere le regole architetturali centrali e quelle locali del repository;
2. rispettare i confini di responsabilità;
3. non inventare nuovi eventi, endpoint o modelli fuori dai contratti;
4. non importare direttamente un altro MFE;
5. utilizzare le librerie condivise versionate;
6. utilizzare gli SDK generati attraverso wrapper locali;
7. non modificare codice generato;
8. aggiungere o aggiornare i test;
9. verificare accessibilità e browser supportati;
10. documentare le decisioni architetturali che introducono nuove dipendenze o pattern.

Ogni repository dovrà contenere istruzioni locali per gli agenti AI coerenti con `case-platform-docs`.

---

## 18. Decisioni consolidate

- La Shell gestisce e compone i MFE.
- I MFE sono applicazioni autonome, non librerie.
- I MFE non si importano direttamente tra loro.
- UI e Frontend Core sono librerie npm versionate.
- Gli SDK TypeScript sono generati dai contratti OpenAPI e pubblicati nel registry npm.
- Ogni MFE utilizza lo SDK del proprio dominio attraverso un wrapper.
- Il `PlatformEventBus` è implementato in `case-platform-frontend-core` usando le API native del browser.
- I contratti degli eventi sono centralizzati, tipizzati e versionati.
- L'Event Bus serve solo per comunicazioni runtime temporanee e disaccoppiate.
- La Shell passa i dati iniziali tramite il contratto di montaggio; i cambiamenti successivi possono essere comunicati tramite Event Bus.
- La compatibilità browser viene definita e verificata automaticamente.
- La configurazione degli ambienti e delle versioni distribuite è governata tramite GitOps.

---

## 19. Riepilogo finale

```text
                  APPLICATION SHELL
                composizione, routing e sessione
                           │
             ┌─────────────┼─────────────┐
             │             │             │
          MFE ANC     MFE SUCCESSIONI  MFE COMMON
             │             │             │
             └─────────────┼─────────────┘
                           │
          Librerie npm e contratti condivisi
          ├── @case-platform/ui
          ├── @case-platform/frontend-core
          ├── @case-platform/frontend-events
          └── @case-platform/*-api-client
```

La Shell governa la composizione. I MFE realizzano funzionalità autonome. Le librerie forniscono codice riutilizzabile. I contratti governano le interfacce. L'Event Bus consente segnalazioni runtime tra componenti indipendenti già caricati.
