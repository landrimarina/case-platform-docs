# Accesso locale ad Argo CD e PostgreSQL

## Scopo

Questo documento raccoglie i comandi e le configurazioni necessari per:

- accedere alla console web di Argo CD;
- collegarsi al PostgreSQL della Case Platform mediante DBeaver;
- recuperare in modo controllato le credenziali;
- interrompere le connessioni temporanee;
- diagnosticare i problemi più comuni.

Argo CD e PostgreSQL non sono esposti pubblicamente. L'accesso dalla macchina locale avviene mediante `kubectl port-forward`.

## Prerequisiti

Prima di procedere verificare:

- k3s in esecuzione;
- `kubectl` configurato per il cluster locale;
- pod Argo CD in stato `Running`;
- cluster PostgreSQL in stato healthy;
- DBeaver installato sulla macchina Windows.

Verifica generale:

```bash
kubectl get nodes
kubectl get pods -A
```

## Accesso alla console Argo CD

### Verifica dei pod

Controllare che i pod Argo CD siano in esecuzione:

```bash
kubectl get pods -n argocd
```

Il pod `argocd-server` deve risultare `Running`.

### Apertura del port-forward

Eseguire in un terminale Ubuntu:

```bash
kubectl port-forward \
  service/argocd-server \
  -n argocd \
  8080:443
```

Risultato atteso:

```text
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Il terminale deve rimanere aperto per tutta la durata della sessione.

### Apertura della console

Aprire nel browser:

```text
https://localhost:8080
```

Il browser può mostrare un avviso relativo al certificato locale. Nel laboratorio è possibile proseguire perché il collegamento è effettuato verso `localhost` mediante port-forward.

### Credenziali iniziali

Nome utente:

```text
admin
```

Recuperare la password amministrativa iniziale:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath='{.data.password}' \
  | base64 --decode
echo
```

La password visualizzata è sensibile e non deve essere copiata in Git, documentazione, log o chat.

Il Secret iniziale può non essere più presente se la password amministrativa è stata modificata e il Secret è stato successivamente rimosso.

### Application Argo CD

Elencare le Application:

```bash
kubectl get applications -n argocd
```

Risultato indicativo:

```text
case-platform-bootstrap
cloudnative-pg-operator
case-platform-postgresql
```

Mostrare informazioni dettagliate sulla root Application:

```bash
kubectl describe application case-platform-bootstrap -n argocd
```

Forzare da terminale un Hard Refresh:

```bash
kubectl annotate application case-platform-bootstrap \
  -n argocd \
  argocd.argoproj.io/refresh=hard \
  --overwrite
```

### Refresh e Sync dalla console

Per rileggere il repository:

1. selezionare l'Application;
2. premere **Refresh**;
3. scegliere **Hard** quando è necessario invalidare la cache;
4. selezionare l'Application;
5. confermare con **Refresh**.

Se l'Application passa a `OutOfSync`:

1. premere **Sync**;
2. verificare le risorse da applicare;
3. premere **Synchronize**.

Principali stati:

| Stato | Significato |
| --- | --- |
| `Synced` | Lo stato del cluster coincide con Git |
| `OutOfSync` | Argo CD rileva differenze rispetto a Git |
| `Healthy` | Le risorse risultano operative |
| `Progressing` | Creazione o aggiornamento in corso |
| `Degraded` | Una o più risorse presentano problemi |

### Chiusura della connessione

Nel terminale in cui è attivo il port-forward premere:

```text
Ctrl+C
```

Dopo la chiusura, `https://localhost:8080` non sarà più raggiungibile.

## Accesso a PostgreSQL con DBeaver

### Verifica del cluster PostgreSQL

Controllare lo stato del cluster:

```bash
kubectl get cluster case-platform-postgresql \
  -n case-platform-data \
  -o wide
```

Controllare pod, Service e storage:

```bash
kubectl get pods -n case-platform-data
kubectl get services -n case-platform-data
kubectl get pvc -n case-platform-data
```

Il risultato atteso comprende:

- pod `case-platform-postgresql-1` in stato `Running`;
- Service `case-platform-postgresql-rw`;
- PVC da `10Gi` in stato `Bound`.

### Service read-write

DBeaver deve collegarsi al Service:

```text
case-platform-postgresql-rw
```

Il suffisso `rw` identifica il Service che indirizza le connessioni verso l'istanza PostgreSQL primaria, abilitata alla lettura e alla scrittura.

### Apertura del port-forward

Eseguire in un terminale Ubuntu:

```bash
kubectl port-forward \
  service/case-platform-postgresql-rw \
  -n case-platform-data \
  15432:5432
```

Risultato atteso:

```text
Forwarding from 127.0.0.1:15432 -> 5432
Forwarding from [::1]:15432 -> 5432
```

La porta locale `15432` viene utilizzata per evitare conflitti con eventuali istanze PostgreSQL o container Docker già associati alla porta standard `5432`.

Il terminale deve rimanere aperto durante l'utilizzo di DBeaver.

### Verifica del Secret

Elencare i Secret senza visualizzarne i valori:

```bash
kubectl get secrets -n case-platform-data
```

Mostrare metadati e chiavi del Secret applicativo:

```bash
kubectl describe secret case-platform-postgresql-app \
  -n case-platform-data
```

CloudNativePG genera automaticamente il Secret:

```text
case-platform-postgresql-app
```

Il Secret contiene le credenziali del proprietario del database applicativo iniziale.

### Recupero dello username

```bash
kubectl get secret case-platform-postgresql-app \
  -n case-platform-data \
  -o jsonpath='{.data.username}' \
  | base64 --decode
echo
```

Risultato previsto:

```text
keycloak
```

### Recupero della password

```bash
kubectl get secret case-platform-postgresql-app \
  -n case-platform-data \
  -o jsonpath='{.data.password}' \
  | base64 --decode
echo
```

La password non deve essere copiata in file versionati, documenti, log o chat.

### Recupero del nome del database

```bash
kubectl get secret case-platform-postgresql-app \
  -n case-platform-data \
  -o jsonpath='{.data.dbname}' \
  | base64 --decode
echo
```

Risultato previsto:

```text
keycloak
```

### Configurazione DBeaver

In DBeaver:

1. creare una nuova connessione;
2. selezionare PostgreSQL;
3. scegliere l'autenticazione tramite username e password;
4. inserire i parametri seguenti.

| Campo | Valore |
| --- | --- |
| Host | `localhost` |
| Porta | `15432` |
| Database | `keycloak` |
| Username | `keycloak` |
| Password | Valore recuperato dal Secret |

URL JDBC risultante:

```text
jdbc:postgresql://localhost:15432/keycloak
```

Per il collegamento cifrato, impostare tra le proprietà del driver:

```text
sslmode=require
```

Premere **Test Connection**. Al primo utilizzo DBeaver può richiedere il download del driver PostgreSQL.

### Stato iniziale del database

Prima dell'installazione di Keycloak, il database contiene lo schema standard ma non le tabelle applicative:

```text
keycloak
└── Schemas
    └── public
```

Al primo avvio Keycloak:

1. si collega al database;
2. verifica la versione dello schema;
3. esegue le proprie migrazioni;
4. crea tabelle, indici e vincoli.

Le tabelle Keycloak non devono essere create o modificate manualmente.

### Chiusura della connessione

Chiudere o disconnettere la connessione in DBeaver.

Nel terminale che esegue il port-forward premere:

```text
Ctrl+C
```

La porta locale `15432` non sarà più inoltrata verso PostgreSQL.

## Esecuzione contemporanea dei port-forward

Per utilizzare contemporaneamente Argo CD e DBeaver sono necessari due terminali distinti.

Terminale 1:

```bash
kubectl port-forward \
  service/argocd-server \
  -n argocd \
  8080:443
```

Terminale 2:

```bash
kubectl port-forward \
  service/case-platform-postgresql-rw \
  -n case-platform-data \
  15432:5432
```

Accessi risultanti:

| Componente | Accesso locale |
| --- | --- |
| Argo CD | `https://localhost:8080` |
| PostgreSQL | `localhost:15432` |

## Problemi comuni

### Connection refused

Possibili cause:

- port-forward non attivo;
- terminale del port-forward chiuso;
- pod non in stato `Running`;
- nome del Service errato;
- porta locale diversa da quella configurata in DBeaver.

Verifiche:

```bash
kubectl get pods -n case-platform-data
kubectl get services -n case-platform-data
```

### Porta locale già occupata

Se la porta `15432` è già utilizzata, scegliere un'altra porta locale:

```bash
kubectl port-forward \
  service/case-platform-postgresql-rw \
  -n case-platform-data \
  25432:5432
```

Aggiornare DBeaver utilizzando:

```text
Porta: 25432
```

### Password authentication failed

Recuperare nuovamente username e password dal Secret e controllare di non aver copiato spazi o ritorni a capo.

```bash
kubectl get secret case-platform-postgresql-app \
  -n case-platform-data \
  -o jsonpath='{.data.username}' \
  | base64 --decode
echo

kubectl get secret case-platform-postgresql-app \
  -n case-platform-data \
  -o jsonpath='{.data.password}' \
  | base64 --decode
echo
```

### Database inesistente

Verificare il nome memorizzato nel Secret:

```bash
kubectl get secret case-platform-postgresql-app \
  -n case-platform-data \
  -o jsonpath='{.data.dbname}' \
  | base64 --decode
echo
```

### Pod PostgreSQL non operativo

```bash
kubectl describe cluster case-platform-postgresql \
  -n case-platform-data

kubectl get pods -n case-platform-data

kubectl describe pod case-platform-postgresql-1 \
  -n case-platform-data

kubectl get events \
  -n case-platform-data \
  --sort-by=.metadata.creationTimestamp
```

### Console Argo CD non raggiungibile

Verificare il pod e il Service:

```bash
kubectl get pods -n argocd
kubectl get service argocd-server -n argocd
```

Riavviare il port-forward:

```bash
kubectl port-forward \
  service/argocd-server \
  -n argocd \
  8080:443
```

## Regole di sicurezza

- Utilizzare il port-forward solo per accessi locali e temporanei.
- Non trasformare PostgreSQL in un Service pubblico per agevolare DBeaver.
- Non inserire credenziali decodificate nella documentazione.
- Non condividere screenshot che mostrino password o token.
- Interrompere i port-forward quando non sono necessari.
- In produzione utilizzare gli strumenti di accesso e auditing previsti dall'ICT.
- La memorizzazione della password in DBeaver deve rispettare le regole di sicurezza della postazione.
