# Configurazione Keycloak per PGCC BDO

## Obiettivo

Configurare Keycloak come sistema di gestione delle identità e degli accessi per l'applicazione PGCC BDO, mantenendo il realm indipendente dalla singola applicazione e delegando al BFF la gestione del flusso di autenticazione.

## Architettura di autenticazione

Il frontend React non dialoga direttamente con Keycloak. Il login è gestito dal BFF tramite OpenID Connect Authorization Code Flow; il BFF mantiene la sessione applicativa e può esporla al browser mediante cookie `HttpOnly`.

```text
Browser React → BFF PGCC → Keycloak
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

## Client PGCC

| Impostazione | Valore |
| --- | --- |
| Client type | `OpenID Connect` |
| Client ID | `pgcc-bff` |
| Name | `PGCC BFF` |
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

I ruoli sono definiti come **Client Roles** del client `pgcc-bff`.

| Ruolo | Utilizzo |
| --- | --- |
| `PGCC_OPERATOR` | Accesso alle funzionalità operative di PGCC |
| `PGCC_ADMIN` | Accesso alle funzionalità amministrative di PGCC |

`PGCC_ADMIN` è configurato come ruolo composito e include `PGCC_OPERATOR`. Un amministratore acquisisce quindi automaticamente anche le autorizzazioni operative.

## Gruppi

| Gruppo | Ruolo associato |
| --- | --- |
| `pgcc-operators` | `PGCC_OPERATOR` |
| `pgcc-admins` | `PGCC_ADMIN` |

I ruoli vengono assegnati ai gruppi e non direttamente agli utenti. Questa configurazione prepara la futura integrazione con Active Directory, nella quale i gruppi aziendali potranno essere mappati sui gruppi o sui ruoli applicativi.

## Utenti di test

| Utente | Gruppo |
| --- | --- |
| `pgcc.operator` | `pgcc-operators` |
| `pgcc.admin` | `pgcc-admins` |

Le password sono impostate come non temporanee. L'utente amministratore appartiene soltanto a `pgcc-admins`, poiché il ruolo composito `PGCC_ADMIN` include già `PGCC_OPERATOR`.

## Contenuto del token

Il client scope predefinito `roles` è assegnato al client `pgcc-bff`. I ruoli applicativi sono pubblicati nell'access token nel claim:

```text
resource_access.pgcc-bff.roles
```

Token atteso per `pgcc.operator`:

```json
{
  "preferred_username": "pgcc.operator",
  "resource_access": {
    "pgcc-bff": {
      "roles": [
        "PGCC_OPERATOR"
      ]
    }
  }
}
```

Token atteso per `pgcc.admin`:

```json
{
  "preferred_username": "pgcc.admin",
  "resource_access": {
    "pgcc-bff": {
      "roles": [
        "PGCC_ADMIN",
        "PGCC_OPERATOR"
      ]
    }
  }
}
```

Se l'indirizzo email è valorizzato nell'utente, il token contiene anche il claim `email`.

## Configurazione finale

```text
Realm: case-platform
└── Client: pgcc-bff
    ├── Ruolo: PGCC_OPERATOR
    ├── Ruolo composito: PGCC_ADMIN
    │   └── include PGCC_OPERATOR
    ├── Gruppo: pgcc-operators
    │   └── PGCC_OPERATOR
    └── Gruppo: pgcc-admins
        └── PGCC_ADMIN
```
