# Continuous Integration con GitHub Actions

**Stato:** Architettura prevista  
**Ultimo aggiornamento:** 23 luglio 2026

## 1. Scopo

Questo documento descrive il ruolo di GitHub Actions nel processo di Continuous Integration della Case Platform.

La guida operativa per creare, eseguire e diagnosticare i workflow sarà mantenuta separatamente.

## 2. Responsabilità

GitHub Actions governa le attività automatiche avviate da modifiche al codice:

- validazione;
- lint;
- test;
- build;
- controlli di sicurezza;
- generazione degli artefatti;
- costruzione delle immagini;
- pubblicazione nei registry;
- aggiornamento controllato del repository GitOps.

GitHub Actions non distribuisce direttamente le applicazioni nel cluster. La distribuzione è responsabilità di Argo CD.

## 3. Separazione CI/CD

```text
Continuous Integration
GitHub Actions
    ↓
build, test e pubblicazione
    ↓
aggiornamento case-platform-gitops
    ↓
Continuous Delivery
Argo CD
    ↓
sincronizzazione del cluster
```

| Ambito | Componente |
| --- | --- |
| Continuous Integration | GitHub Actions |
| Source of truth del deployment | `case-platform-gitops` |
| Continuous Delivery | Argo CD |
| Runtime | Kubernetes/k3s |

## 4. Eventi di attivazione

I workflow possono essere attivati da:

- apertura o aggiornamento di una pull request;
- push su `develop`;
- push o merge su `main`;
- creazione di un tag;
- esecuzione manuale controllata;
- workflow riusabili richiamati da altri repository.

Ogni evento deve eseguire soltanto le attività coerenti con il relativo livello del ciclo di vita.

## 5. Pipeline delle pull request

Una pull request deve attivare controlli senza produrre un rilascio:

```text
Pull Request
→ validazione struttura
→ lint
→ test unitari
→ test di contratto
→ build
→ scansioni di sicurezza
→ esito della quality gate
```

Il merge deve essere consentito soltanto dopo il superamento dei controlli obbligatori.

## 6. Pipeline di `develop`

Il merge su `develop` produce artefatti destinati all’ambiente di sviluppo/integrazione:

```text
Merge su develop
→ test completi
→ build
→ immagine versionata
→ pubblicazione nel registry
→ aggiornamento della configurazione DEV
→ Argo CD sincronizza il cluster
```

Gli artefatti devono essere identificabili e riconducibili alla commit sorgente.

## 7. Pipeline di `main`

Il merge su `main` rappresenta una versione approvata:

```text
Merge su main
→ validazione finale
→ produzione artefatti rilasciabili
→ tag di versione
→ pubblicazione
→ aggiornamento configurazione di produzione
→ promozione tramite GitOps
```

La pipeline non deve ricostruire in modo differente un artefatto già validato. Quando possibile, deve promuovere lo stesso artefatto tra gli ambienti.

## 8. Pipeline per tipologia di repository

### Frontend condiviso

Per `case-platform-frontend`:

- lint TypeScript;
- test unitari;
- test dei componenti;
- test di accessibilità;
- build della Shell;
- build del MFE Common;
- build e pubblicazione delle librerie condivise;
- test di integrazione tra Shell e MFE.

### Repository full-stack di dominio

Per `case-platform-anc` e gli altri domini:

- validazione del contratto OpenAPI;
- generazione degli SDK;
- test frontend;
- build del MFE;
- test backend;
- build del backend;
- test di contratto;
- test di integrazione;
- test end-to-end;
- costruzione separata delle immagini frontend e backend.

### BFF

Per `case-platform-bff`:

- test unitari;
- test OIDC;
- test di sessione;
- test di sicurezza;
- build;
- costruzione dell’immagine;
- verifica della configurazione.

### Contratti

Per `case-platform-contracts`:

- validazione OpenAPI;
- applicazione delle regole comuni;
- verifica dei breaking change;
- generazione degli SDK;
- pubblicazione dei pacchetti versionati.

### GitOps

Per `case-platform-gitops`:

- validazione YAML;
- render dei chart Helm;
- validazione Kustomize;
- controllo delle policy;
- verifica dei riferimenti alle immagini;
- eventuale dry-run contro il cluster di test.

### Documentazione

Per `case-platform-docs`:

- lint Markdown;
- verifica dei collegamenti;
- controllo della struttura;
- eventuale generazione del sito documentale.

## 9. Artefatti

Le pipeline possono produrre:

- pacchetti npm;
- artefatti Maven;
- SDK TypeScript;
- SDK o modelli Java;
- immagini OCI;
- report di test;
- report di sicurezza;
- documentazione generata.

Ogni artefatto deve avere:

- versione esplicita;
- commit sorgente;
- pipeline che lo ha generato;
- checksum o identificativo immutabile;
- destinazione di pubblicazione.

## 10. Versionamento

Le librerie e gli SDK utilizzano versionamento semantico:

```text
MAJOR.MINOR.PATCH
```

- `MAJOR`: modifica incompatibile;
- `MINOR`: nuova funzionalità compatibile;
- `PATCH`: correzione compatibile.

Le immagini applicative devono utilizzare tag immutabili. Il tag `latest` non deve essere utilizzato nei manifest GitOps.

## 11. Sicurezza

Le pipeline devono applicare:

- principio del least privilege;
- protezione dei branch;
- approvazione delle pull request;
- dipendenze versionate;
- scansione delle vulnerabilità;
- secret gestiti tramite GitHub Secrets o meccanismi equivalenti;
- divieto di stampare secret nei log;
- permessi GitHub Actions dichiarati esplicitamente;
- separazione tra credenziali di build e credenziali di deployment.

GitHub Actions non deve ricevere credenziali amministrative permanenti del cluster quando il deployment è affidato ad Argo CD.

## 12. Workflow riusabili

Per evitare duplicazioni, i workflow comuni potranno essere definiti come workflow riusabili:

```text
validate-java
test-java
build-java
validate-frontend
test-frontend
build-frontend
validate-openapi
build-container-image
update-gitops
```

I repository di dominio li richiameranno passando configurazioni esplicite.

## 13. Relazione con gli agenti AI

Gli agenti AI possono modificare codice e workflow nel rispetto delle regole del repository, ma non devono:

- disattivare controlli per far passare una pipeline;
- ampliare permessi senza motivazione;
- inserire secret nel codice;
- usare versioni mobili non controllate;
- modificare artefatti generati manualmente;
- eseguire rilasci senza l’autorizzazione prevista.

## 14. Stato di implementazione

La struttura dei repository è stata predisposta con directory `.github/workflows`.

I workflow applicativi devono ancora essere implementati e saranno introdotti progressivamente insieme ai primi vertical slice.

## 15. Documentazione correlata

- `architecture/platform/gitops-argocd.md`
- `development/local-development-environment.md`
- future guide operative in `development/guides`

