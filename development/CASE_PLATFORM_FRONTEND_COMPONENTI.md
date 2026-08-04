# Case Platform Frontend

## Componenti, utilizzo e installazione

## 1. Obiettivo del documento

Questo documento descrive i componenti utilizzati per realizzare il frontend della **Case Platform**, il ruolo di ciascun componente e le relative modalità di installazione.

La soluzione frontend è organizzata come **monorepo** e comprenderà:

- una **Shell**, responsabile della struttura generale dell'interfaccia;
- uno o più **Micro Frontend (MFE) di dominio**, ad esempio ANC e Successioni;
- un **MFE comune**, per funzionalità applicative trasversali;
- librerie condivise per interfaccia grafica e servizi tecnici frontend.

## 2. Vista sintetica dei componenti

| Componente | Utilizzo | Ambito di installazione |
|---|---|---|
| WSL2 | Ambiente Linux di sviluppo su Windows | Una sola volta sulla postazione |
| NVM | Gestione delle versioni di Node.js | Una sola volta nell'ambiente WSL |
| Node.js | Runtime degli strumenti frontend | Una sola volta tramite NVM |
| npm | Gestione dipendenze, workspace e comandi di build | Installato insieme a Node.js |
| npm workspaces | Gestione del monorepo | Configurato una sola volta nella radice |
| React | Realizzazione delle interfacce utente | Dipendenza di ogni applicazione frontend |
| TypeScript | Tipizzazione statica del codice | Dipendenza di sviluppo di ogni applicazione |
| Vite | Avvio locale e build delle applicazioni | Dipendenza di sviluppo di ogni applicazione |
| ESLint | Controllo qualità e uniformità del codice | Dipendenza di sviluppo di ogni applicazione |
| Material UI | Componenti grafici e base del Design System | Condiviso tramite la libreria `ui` |
| Emotion | Motore di stile utilizzato da Material UI | Dipendenza associata a Material UI |
| NGINX | Pubblicazione dei file statici compilati | Ambiente di esecuzione/deployment |

## 3. Struttura del monorepo

La radice del repository contiene un unico progetto npm con più workspace:

```text
case-platform-frontend/
├── apps/
│   ├── shell/
│   └── mfe-common/
├── libs/
│   ├── ui/
│   └── frontend-core/
├── tests/
├── tools/
├── package.json
└── package-lock.json
```

I futuri frontend di dominio potranno essere aggiunti in `apps`, ad esempio:

```text
apps/mfe-anc/
apps/mfe-successioni/
```

### Responsabilità principali

| Modulo | Responsabilità |
|---|---|
| `apps/shell` | Layout generale, header, menu, sessione, configurazione, navigazione e composizione dei MFE |
| `apps/mfe-common` | Funzionalità applicative comuni, come profilo, notifiche e ricerca globale |
| `apps/mfe-<dominio>` | Pagine, casi d'uso e funzionalità specifiche del dominio |
| `libs/ui` | Tema, token grafici, icone e componenti visuali riutilizzabili |
| `libs/frontend-core` | HTTP, eventi, configurazione, logging, correlazione ed error handling condivisi |
| `tools` | Script e generatori per creare nuovi MFE secondo lo standard della piattaforma |

## 4. Componenti dell'ambiente di sviluppo

### 4.1 WSL2

**Utilizzo**

WSL2 consente di usare un ambiente Linux direttamente su Windows. Il codice, Node.js, npm e gli altri strumenti del progetto vengono eseguiti all'interno di WSL, evitando di mescolare installazioni Windows e Linux.

**Installazione**

Da PowerShell eseguita come amministratore:

```powershell
wsl --install
```

Dopo il riavvio, verificare da PowerShell:

```powershell
wsl --status
```

Nel progetto è utilizzata una distribuzione Ubuntu in WSL2.

### 4.2 NVM

**Utilizzo**

NVM, Node Version Manager, installa e gestisce le versioni di Node.js per il singolo utente Linux. Permette di aggiornare o cambiare versione senza utilizzare l'installazione Node di Windows.

**Installazione**

L'installazione va eseguita nel terminale WSL seguendo lo script ufficiale di NVM. Dopo l'installazione si ricarica la shell e si verifica:

```bash
command -v nvm
nvm --version
```

Per installare e selezionare una versione di Node.js:

```bash
nvm install 24
nvm use 24
nvm alias default 24
```

### 4.3 Node.js

**Utilizzo**

Node.js è il runtime che esegue gli strumenti frontend, tra cui npm, Vite, TypeScript ed ESLint. Non è il runtime dell'applicazione distribuita nel browser: serve principalmente durante sviluppo, test e build.

**Installazione**

Node.js viene installato tramite NVM:

```bash
nvm install 24
```

Versione attualmente verificata:

```text
Node.js v24.19.0
```

Verifica:

```bash
command -v node
node --version
```

Il comando deve restituire un percorso interno a WSL, simile a:

```text
/home/<utente>/.nvm/versions/node/v24.19.0/bin/node
```

### 4.4 npm

**Utilizzo**

npm è il package manager del progetto frontend. Il suo ruolo è paragonabile in parte a quello di Maven nel mondo Java:

- legge la configurazione da `package.json`;
- scarica e aggiorna le dipendenze;
- gestisce i workspace del monorepo;
- esegue gli script di sviluppo, build, lint e test;
- registra le versioni effettive in `package-lock.json`.

**Installazione**

npm viene installato insieme a Node.js. Non richiede un'installazione separata.

Versione attualmente verificata:

```text
npm 11.17.0
```

Verifica:

```bash
command -v npm
npm --version
```

## 5. Configurazione del monorepo npm

### 5.1 Inizializzazione

Dalla radice del repository:

```bash
npm init -y
```

### 5.2 Configurazione dei workspace

Il `package.json` principale deve contenere almeno:

```json
{
  "name": "case-platform-frontend",
  "private": true,
  "type": "module",
  "workspaces": [
    "apps/*",
    "libs/*"
  ]
}
```

La proprietà `private` impedisce la pubblicazione accidentale dell'intero monorepo sul registry npm.

I workspace consentono di gestire applicazioni e librerie con un'unica installazione dalla radice:

```bash
npm install
```

Le dipendenze vengono normalmente centralizzate nella directory `node_modules` della radice e le versioni vengono registrate nel solo `package-lock.json` principale.

## 6. Componenti applicativi

### 6.1 React

**Utilizzo**

React è la libreria utilizzata per realizzare le interfacce utente della Shell, del MFE comune e dei MFE di dominio.

Ogni applicazione frontend avrà un proprio punto di ingresso e potrà essere:

- avviata autonomamente durante lo sviluppo;
- compilata separatamente;
- integrata nella Shell come Micro Frontend.

**Installazione in un workspace**

```bash
npm install react react-dom --workspace=<nome-workspace>
```

Per la Shell:

```bash
npm install react react-dom --workspace=case-platform-shell
```

Versione attualmente configurata nella Shell:

```text
React 19.2.8
```

### 6.2 TypeScript

**Utilizzo**

TypeScript aggiunge la tipizzazione statica a JavaScript. Consente di individuare errori durante lo sviluppo e rende più sicuri i contratti tra componenti, librerie e servizi backend.

**Installazione in un workspace**

```bash
npm install --save-dev typescript @types/react @types/react-dom @types/node \
  --workspace=<nome-workspace>
```

La configurazione è distribuita nei file:

```text
tsconfig.json
tsconfig.app.json
tsconfig.node.json
```

### 6.3 Vite

**Utilizzo**

Vite gestisce:

- il server locale di sviluppo;
- l'aggiornamento automatico dell'interfaccia durante le modifiche;
- la trasformazione del codice TypeScript e React;
- la produzione dei file statici ottimizzati nella cartella `dist`.

Vite non viene eseguito nell'ambiente finale: produce gli asset che saranno pubblicati tramite NGINX.

**Installazione in un workspace React**

```bash
npm install --save-dev vite @vitejs/plugin-react --workspace=<nome-workspace>
```

**Comandi principali**

```bash
npm run dev --workspace=case-platform-shell
npm run build --workspace=case-platform-shell
npm run preview --workspace=case-platform-shell
```

### 6.4 ESLint

**Utilizzo**

ESLint controlla automaticamente il codice e segnala errori, costrutti rischiosi e violazioni delle convenzioni definite per il progetto.

La configurazione della Shell si trova in:

```text
apps/shell/eslint.config.js
```

**Installazione in un workspace React e TypeScript**

```bash
npm install --save-dev \
  eslint @eslint/js globals typescript-eslint \
  eslint-plugin-react-hooks eslint-plugin-react-refresh \
  --workspace=<nome-workspace>
```

**Esecuzione**

```bash
npm run lint --workspace=case-platform-shell
```

### 6.5 Material UI

**Utilizzo**

Material UI fornisce componenti React accessibili e coerenti, come pulsanti, menu, tabelle, campi di input, finestre di dialogo e strutture di layout.

Nella fase iniziale è stato installato nella Shell per verificare la baseline. Il tema e i componenti comuni saranno successivamente collocati in `libs/ui`, in modo che Shell e MFE di dominio utilizzino lo stesso Design System senza duplicazioni.

**Installazione nella Shell**

```bash
npm install \
  @mui/material @mui/icons-material \
  @emotion/react @emotion/styled \
  --workspace=case-platform-shell
```

Versione attualmente configurata:

```text
Material UI 9.2.0
```

### 6.6 Emotion

**Utilizzo**

Emotion è il motore di gestione degli stili utilizzato da Material UI. Consente a Material UI di applicare temi e stili dinamici ai componenti React.

Non viene utilizzato come componente applicativo autonomo: è una dipendenza necessaria di Material UI.

**Installazione**

```bash
npm install @emotion/react @emotion/styled --workspace=<nome-workspace>
```

### 6.7 Tema della piattaforma

**Utilizzo**

Il tema centralizza colori, tipografia, spaziature e caratteristiche visuali. Attualmente la baseline si trova in:

```text
apps/shell/src/styles/theme.ts
```

Il tema viene applicato alla Shell tramite `ThemeProvider` e `CssBaseline` nel file:

```text
apps/shell/src/main.tsx
```

Esempio:

```tsx
<ThemeProvider theme={platformTheme}>
  <CssBaseline />
  <App />
</ThemeProvider>
```

Quando `libs/ui` sarà configurata come package condiviso, il tema verrà spostato nella libreria e importato da tutte le applicazioni.

## 7. NGINX

**Utilizzo**

L'applicazione React viene eseguita nel browser. Dopo la build, Vite produce file HTML, JavaScript, CSS e asset statici nella cartella `dist`.

NGINX viene utilizzato nell'ambiente di deployment per:

- distribuire i file statici compilati;
- gestire il fallback della navigazione client-side verso `index.html`;
- applicare configurazioni HTTP, caching e header di sicurezza;
- esporre l'applicazione nel container o nel pod Kubernetes.

**Installazione locale opzionale su Ubuntu**

NGINX non è necessario per il normale sviluppo con Vite. Se occorre provarlo localmente:

```bash
sudo apt update
sudo apt install nginx
nginx -v
```

Nel deployment Kubernetes sarà preferibile utilizzare un'immagine container NGINX; l'installazione sul sistema operativo del cluster non è necessaria.

## 8. Comandi operativi della Shell

Dalla radice del repository:

### Installazione delle dipendenze

```bash
npm install
```

### Avvio in sviluppo

```bash
npm run dev --workspace=case-platform-shell
```

La Shell è normalmente disponibile su:

```text
http://localhost:5173
```

### Controllo del codice

```bash
npm run lint --workspace=case-platform-shell
```

### Build di produzione

```bash
npm run build --workspace=case-platform-shell
```

L'output viene generato in:

```text
apps/shell/dist/
```

La directory `dist` è generata e non deve essere versionata in Git.

## 9. Cosa si ripete per un nuovo MFE di dominio

Ogni nuovo MFE avrà:

- un proprio `package.json` di workspace;
- una configurazione Vite;
- una configurazione TypeScript;
- una configurazione ESLint;
- un punto di ingresso React;
- gli script `dev`, `build`, `lint` e test;
- una modalità standalone per lo sviluppo;
- una configurazione per l'integrazione nella Shell.

Non devono invece essere reinstallati o duplicati:

- WSL2, NVM, Node.js e npm;
- la configurazione del monorepo;
- il tema e i componenti grafici condivisi;
- i servizi tecnici contenuti in `frontend-core`;
- header, menu, sessione e navigazione generale della Shell.

Quando la baseline sarà consolidata, un generatore in `tools/generators` creerà automaticamente la struttura standard dei nuovi MFE.

## 10. File da versionare e file da escludere

### Da versionare

```text
package.json
package-lock.json
apps/*/package.json
apps/*/src/
apps/*/public/
apps/*/vite.config.ts
apps/*/tsconfig*.json
apps/*/eslint.config.js
libs/
```

### Da escludere tramite `.gitignore`

```text
node_modules/
dist/
coverage/
.env
.env.local
*:Zone.Identifier
```

Il file `package-lock.json` deve essere versionato: garantisce che gli ambienti di sviluppo e la pipeline CI installino le stesse versioni delle dipendenze.

## 11. Stato attuale della baseline

La baseline della Shell comprende attualmente:

- Node.js e npm installati in WSL tramite NVM;
- monorepo npm con workspace `apps/*` e `libs/*`;
- Shell React con TypeScript e Vite;
- controllo del codice con ESLint;
- Material UI ed Emotion;
- tema iniziale della piattaforma;
- build e lint completati correttamente.

I prossimi passi riguardano la realizzazione del layout della Shell, la configurazione delle librerie condivise e, successivamente, l'integrazione dei Micro Frontend.
