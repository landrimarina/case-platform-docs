# ADR-0001 — Strategia BFF: unico di piattaforma vs uno per dominio

**Stato:** Accettato
**Data:** 2026-09-16
**Area:** Backend / Autenticazione

## Contesto

La Case Platform adotta un'architettura a microfrontend: una Shell (host) carica a runtime N MFE di dominio (oggi PGCC, in futuro ANC e altri). L'autenticazione è delegata a un **BFF** che agisce da client OIDC verso Keycloak (realm `case-platform`), mantiene la sessione applicativa (cookie `HttpOnly`) ed espone alla Shell solo dati derivati tramite `GET /api/session/me` (`username`, `roles`). Il frontend non dialoga mai direttamente con Keycloak.

Con l'ingresso di nuovi domini si deve decidere **quanti BFF** avere e **come modellare i client OIDC e i ruoli**.

### Ambiguità attuale da sciogliere

Esiste una tensione nel naming che questo ADR risolve:

- Il repository è `case-platform-bff` → suggerisce un **BFF unico** di piattaforma.
- Il client OIDC è `pgcc-bff` e il claim dei ruoli è `resource_access/pgcc-bff/roles` → suggerisce un **BFF per dominio**.

Oggi la configurazione funziona solo per PGCC. Aggiungere ANC richiede una decisione esplicita, altrimenti i ruoli `ANC_*` non finirebbero nel claim letto dal BFF.

## Problema

Come strutturare il livello BFF e i client Keycloak per supportare più domini mantenendo:

- Single Sign-On e un'unica sessione lato utente;
- autonomia di deploy dei domini;
- un BFF sottile (solo autenticazione/sessione e proxy, senza business logic);
- autorizzazione reale sempre sul backend.

## Opzioni considerate

### Opzione A — BFF unico di piattaforma

Un solo BFF per tutti i domini; in Keycloak un unico client (es. `case-platform-bff`) con tutti i ruoli (`PGCC_*`, `ANC_*`).

```text
Shell ─► BFF unico ─► Keycloak (client: case-platform-bff)
             ├─ /api/pgcc → pgcc-backend
             └─ /api/anc  → anc-backend
```

**Pro**
- Una sola sessione / un solo cookie: SSO naturale, login una volta sola.
- Meno infrastruttura: un container, un client Keycloak, una config OIDC.
- `role-claim-path` semplice: tutti i ruoli in un unico client.
- Landing e menu per ruolo banali: la Shell ha già tutti i ruoli in `/api/session/me`.

**Contro**
- Accoppiamento: tutti i domini dipendono da un componente condiviso; un deploy del BFF impatta tutti.
- Rischio "god service" se il BFF accumula logica di più domini.
- Contraddice parzialmente l'autonomia "repo full-stack per dominio".

### Opzione B — Un BFF per dominio

Ogni dominio ha il proprio BFF e il proprio client OIDC (`pgcc-bff`, `anc-bff`), ciascuno con i propri ruoli.

```text
Shell ─► pgcc-bff ─► Keycloak (client: pgcc-bff)  ─► pgcc-backend
     └─► anc-bff  ─► Keycloak (client: anc-bff)   ─► anc-backend
```

**Pro**
- Autonomia piena per dominio: deploy, scaling e sicurezza indipendenti.
- Isolamento: un problema sul BFF ANC non tocca PGCC.
- Ruoli segregati per client: un dominio non "vede" i ruoli dell'altro.

**Contro**
- Sessione multipla: ogni BFF ha il proprio cookie/callback → serve orchestrare l'SSO (mitigato dal cookie SSO di Keycloak, ma la Shell deve aggregare più `session/me`).
- Più infrastruttura: N client, N config OIDC, N container, più route su APISIX.
- La Shell deve sapere quale BFF interrogare per sessione/ruoli di ciascun dominio; il landing per ruolo richiede di aggregare più sorgenti.

## Decisione

Si adotta un modello **ibrido basato sull'Opzione A**:

> **Un BFF unico "di sessione/piattaforma"** (gestione OIDC, cookie, `/api/session/me`, ruoli di tutti i domini) + **backend di dominio separati** (`pgcc-backend`, `anc-backend`) dietro il gateway.

L'autonomia dei domini resta dove conta davvero — nei **backend di dominio** e nei **MFE**, che restano indipendenti — mentre l'autenticazione e la sessione rimangono centralizzate per garantire SSO e un'unica UX nella Shell. Il BFF resta sottile: solo autenticazione/sessione e proxy verso i backend di dominio, nessuna business logic.

## Conseguenze

### Modifiche necessarie

1. Rinominare il client OIDC da `pgcc-bff` a `case-platform-bff`. **(fatto — `set-env-bff.sh`, doc)**
2. Registrare in Keycloak i ruoli di tutti i domini sotto quel client (`PGCC_*`, `ANC_*`). **(azione manuale in console Keycloak)**
3. Aggiornare `quarkus.oidc.roles.role-claim-path=resource_access/case-platform-bff/roles`. **(già dinamico: usa `${quarkus.oidc.client-id}`)**
4. Aggiornare i redirect URI e la documentazione [configurazione-keycloak-piattaforma.md](../../development/configurazione-keycloak-piattaforma.md). **(doc aggiornata; redirect URI da aggiornare in console Keycloak)**

### Vincoli permanenti

- L'autorizzazione reale resta sul backend/BFF: ogni `/api/<dominio>/**` verifica i ruoli lato server. Il filtro per ruolo nel frontend è solo UX, non un controllo di sicurezza.
- Il token OIDC non deve mai arrivare al browser: la sessione è del BFF (cookie `HttpOnly`); il frontend vede solo dati derivati.
- Il BFF non deve contenere business logic di dominio.

### Quando rivalutare (passaggio all'Opzione B)

Se in futuro i domini avranno requisiti di sicurezza/tenancy realmente separati (es. realm distinti, compliance che impone isolamento) o team che vogliono possedere anche il proprio strato di sessione. In quel caso serve un nuovo ADR che sostituisca questo e definisca una strategia SSO esplicita.
