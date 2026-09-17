# Configurazione Keycloak della Case Platform

## Obiettivo

Configurare Keycloak come sistema di gestione delle identità e degli accessi per l'intera Case Platform, con un **unico realm** e un **unico client BFF** condiviso da tutti i domini (PGCC, ANC, futuri). Il realm resta indipendente dalle singole applicazioni e il flusso di autenticazione è delegato al BFF.

## Architettura di autenticazione

Il frontend React non dialoga direttamente con Keycloak. Il login è gestito dal BFF tramite OpenID Connect Authorization Code Flow; il BFF mantiene la sessione applicativa e la espone al browser mediante cookie `HttpOnly`.

```text
Browser (Shell + MFE) → BFF di piattaforma → Keycloak
```

## Accesso alla console di amministrazione

Il servizio Keycloak installato nel cluster è:

```text
service/case-platform-keycloak-service
Namespace: keycloak
Porte: 8080, 9000
```

Per accedere localmente alla console:

```bash
kubectl port-forward -n keycloak \
  service/case-platform-keycloak-service 8080:8080
```

Console di amministrazione:

```text
http://localhost:8080/admin
```

Le credenziali amministrative iniziali sono conservate nel Secret:

```text
case-platform-keycloak-initial-admin
```

## Realm

| Impostazione | Valore |
| --- | --- |
| Realm name | `case-platform` |
| Enabled | `ON` |

Il realm rappresenta il perimetro comune di identità e sicurezza della piattaforma. Non è specifico di PGCC e potrà contenere client dedicati ad altre applicazioni.

## Client BFF di piattaforma

| Impostazione | Valore |
| --- | --- |
| Client type | `OpenID Connect` |
| Client ID | `case-platform-bff` |
| Name | `Case Platform BFF` |
| Client authentication | `ON` |
| Authorization | `OFF` |
| Standard flow | `ON` |
| Direct access grants | `OFF` |
| Implicit flow | `OFF` |
| Service account roles | `OFF` |
| Require PKCE | `ON` |
| Require DPoP bound tokens | `OFF` |

Il client è riservato perché è il BFF, e non il frontend React, a comunicare con Keycloak utilizzando le proprie credenziali.

### URL per lo sviluppo locale

| Impostazione | Valore |
| --- | --- |
| Root URL | vuoto |
| Home URL | `http://localhost:5173` |
| Valid redirect URIs | `http://localhost:8081/auth/callback` |
| Valid post logout redirect URIs | `http://localhost:5173/` |
| Web origins | `http://localhost:5173` |

Nel BFF Quarkus il callback OIDC corrisponde a:

```properties
quarkus.oidc.authentication.redirect-path=/auth/callback
```

Gli URL esposti tramite APISIX dovranno essere aggiunti quando saranno definiti gli indirizzi dei servizi nel cluster.

## Ruoli applicativi

I ruoli sono definiti come **Client Roles** dell'unico client `case-platform-bff` e sono nominati con il prefisso del dominio. `ADMIN` e `OPERATOR` sono ruoli **indipendenti**: un utente può avere solo `ADMIN`, solo `OPERATOR`, oppure entrambi. Non si usano ruoli compositi, in modo che ogni combinazione sia assegnabile esplicitamente.

| Ruolo | Utilizzo |
| --- | --- |
| `PGCC_OPERATOR` | Funzionalità operative di PGCC |
| `PGCC_ADMIN` | Funzionalità amministrative di PGCC |
| `ANC_OPERATOR` | Funzionalità operative di ANC |
| `ANC_ADMIN` | Funzionalità amministrative di ANC |

> Ogni nuovo dominio aggiunge i propri client role con il prefisso corrispondente (es. `XXX_OPERATOR`, `XXX_ADMIN`) sotto lo stesso client.

## Gruppi

I ruoli vengono assegnati ai gruppi e **non direttamente agli utenti**. Si usa un gruppo per ruolo, così ogni combinazione si ottiene componendo più gruppi. Questa configurazione prepara la futura integrazione con Active Directory, nella quale i gruppi aziendali potranno essere mappati sui gruppi applicativi.

| Gruppo | Ruolo associato |
| --- | --- |
| `pgcc-operators` | `PGCC_OPERATOR` |
| `pgcc-admins` | `PGCC_ADMIN` |
| `anc-operators` | `ANC_OPERATOR` |
| `anc-admins` | `ANC_ADMIN` |

## Utenti di test

Componendo i gruppi si ottiene qualsiasi combinazione di dominio e ruolo:

| Utente | Accesso desiderato | Gruppi | Ruoli nel token |
| --- | --- | --- | --- |
| `x` | PGCC admin | `pgcc-admins` | `PGCC_ADMIN` |
| `y` | PGCC admin + operator | `pgcc-admins`, `pgcc-operators` | `PGCC_ADMIN`, `PGCC_OPERATOR` |
| `z` | ANC admin | `anc-admins` | `ANC_ADMIN` |
| `a` | ANC admin + operator | `anc-admins`, `anc-operators` | `ANC_ADMIN`, `ANC_OPERATOR` |
| `b` | admin su PGCC e ANC | `pgcc-admins`, `anc-admins` | `PGCC_ADMIN`, `ANC_ADMIN` |

Le password sono impostate come non temporanee.

## Contenuto del token

Il client scope predefinito `roles` è assegnato al client `case-platform-bff`. I ruoli applicativi sono pubblicati nell'access token nel claim:

```text
resource_access.case-platform-bff.roles
```

Token atteso per `y` (PGCC admin + operator):

```json
{
  "preferred_username": "y",
  "resource_access": {
    "case-platform-bff": {
      "roles": [
        "PGCC_ADMIN",
        "PGCC_OPERATOR"
      ]
    }
  }
}
```

Token atteso per `b` (admin su entrambi i domini):

```json
{
  "preferred_username": "b",
  "resource_access": {
    "case-platform-bff": {
      "roles": [
        "PGCC_ADMIN",
        "ANC_ADMIN"
      ]
    }
  }
}
```

Se l'indirizzo email è valorizzato nell'utente, il token contiene anche il claim `email`.

## Configurazione finale

```text
Realm: case-platform
└── Client: case-platform-bff
    ├── Ruoli (indipendenti): PGCC_OPERATOR, PGCC_ADMIN, ANC_OPERATOR, ANC_ADMIN
    ├── Gruppo pgcc-operators → PGCC_OPERATOR
    ├── Gruppo pgcc-admins    → PGCC_ADMIN
    ├── Gruppo anc-operators  → ANC_OPERATOR
    └── Gruppo anc-admins     → ANC_ADMIN
```
