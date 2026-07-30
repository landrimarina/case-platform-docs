# Guida operativa a k3s e Kubernetes

## Scopo

Questo documento introduce i concetti Kubernetes utilizzati nella Case Platform e raccoglie i principali comandi operativi per osservare e diagnosticare l'ambiente locale k3s.

Non sostituisce la documentazione di installazione né descrive una configurazione di produzione. L'obiettivo è fornire una guida sintetica per comprendere cosa viene creato nel cluster, quale componente ne è responsabile e come verificarne lo stato.

## Kubernetes e k3s

Kubernetes è una piattaforma per eseguire e amministrare applicazioni containerizzate. Riceve una configurazione dichiarativa e cerca continuamente di mantenere lo stato reale del cluster allineato allo stato desiderato.

k3s è una distribuzione Kubernetes leggera. Offre le principali API e funzionalità Kubernetes riducendo le risorse necessarie, ed è quindi adatta al laboratorio locale della Case Platform.

Nel laboratorio:

```text
Windows
└── WSL 2
    └── Ubuntu
        └── k3s
            ├── componenti Kubernetes
            ├── Argo CD
            ├── CloudNativePG Operator
            ├── PostgreSQL
            └── future componenti della Case Platform
```

k3s utilizza `containerd` per avviare i container. Docker non è necessario come runtime del cluster.

## Cluster e nodo

Il **cluster** è l'ambiente Kubernetes nel suo complesso.

Un **nodo** è una macchina che mette a disposizione CPU, memoria e storage per eseguire i workload. Nel laboratorio k3s è presente un solo nodo, ospitato nella distribuzione Ubuntu di WSL.

Questa configurazione permette di validare installazione e integrazione delle componenti, ma non fornisce alta disponibilità: se la macchina o WSL diventano indisponibili, non esiste un secondo nodo sul quale spostare i workload.

## Risorse principali

### Namespace

Un namespace è una separazione logica delle risorse Kubernetes.

La Case Platform utilizza namespace distinti per evitare di mescolare componenti con responsabilità differenti.

Esempi:

| Namespace | Contenuto |
| --- | --- |
| `kube-system` | Componenti interni di k3s |
| `argocd` | Argo CD e le sue Application |
| `cnpg-system` | CloudNativePG Operator |
| `case-platform-data` | Cluster PostgreSQL della piattaforma |
| `keycloak` | Keycloak |
| `case-platform` | Componenti condivise della piattaforma |
| `case-platform-anc` | Componenti applicative del dominio ANC |

Il namespace non costituisce da solo una separazione di sicurezza completa. L'isolamento viene rafforzato mediante RBAC, NetworkPolicy, Secret e account applicativi dedicati.

### Pod

Il pod è la più piccola unità eseguibile di Kubernetes. Contiene uno o più container che condividono rete e volumi.

Normalmente non si crea né si amministra direttamente un pod. Il pod viene generato e controllato da una risorsa di livello superiore, come:

- Deployment;
- StatefulSet;
- DaemonSet;
- Job;
- un operatore Kubernetes.

Se un pod controllato viene eliminato, il controller tende a ricrearlo per ripristinare lo stato desiderato.

### Deployment

Un Deployment gestisce applicazioni generalmente stateless, cioè componenti che non conservano il proprio stato nel filesystem del container.

Esempi tipici:

- BFF;
- backend applicativi;
- API Gateway;
- operatori Kubernetes;
- frontend serviti da un web server.

Il Deployment governa:

- numero di repliche;
- immagine container;
- aggiornamenti progressivi;
- ricreazione dei pod;
- configurazione delle risorse.

### StatefulSet

Uno StatefulSet gestisce workload che richiedono identità stabile e storage persistente.

È frequentemente utilizzato dai database. Nel caso della Case Platform, la gestione concreta delle risorse PostgreSQL è orchestrata da CloudNativePG.

### Service

Un Service fornisce un indirizzo stabile per raggiungere uno o più pod.

I pod possono essere eliminati e ricreati con indirizzi IP differenti; il Service mantiene invece un nome DNS stabile.

Principali tipi:

| Tipo | Utilizzo |
| --- | --- |
| `ClusterIP` | Accesso interno al cluster |
| `NodePort` | Espone una porta su ciascun nodo |
| `LoadBalancer` | Richiede o simula un bilanciatore esterno |

k3s può creare pod con prefisso `svclb-` per implementare localmente i Service di tipo `LoadBalancer`. Eliminare soltanto un pod `svclb-` non risolve il problema: se il Service esiste ancora, il pod viene ricreato.

### Ingress

Un Ingress descrive regole HTTP e HTTPS per rendere raggiungibili i servizi.

Nel laboratorio Traefik è l'Ingress Controller installato da k3s. Riceve il traffico in ingresso e lo inoltra al Service previsto.

Nel disegno della Case Platform, Traefik espone i punti di accesso del cluster, mentre APISIX governa routing e policy delle API.

### ConfigMap

Una ConfigMap contiene configurazioni non sensibili:

- proprietà applicative;
- file di configurazione;
- valori specifici dell'ambiente.

Non deve essere utilizzata per password, token o certificati privati.

### Secret

Un Secret contiene dati sensibili utilizzati dai workload:

- password;
- token;
- chiavi;
- certificati.

I Secret Kubernetes sono associati a un namespace. Un pod non può montare direttamente un Secret appartenente a un altro namespace.

I valori presenti in un Secret standard sono codificati in Base64, ma non sono automaticamente cifrati in modo equivalente a un secret manager. I Secret non devono essere inseriti in chiaro nel repository Git.

### PersistentVolume e PersistentVolumeClaim

Un PersistentVolume, o PV, rappresenta uno spazio di archiviazione disponibile.

Un PersistentVolumeClaim, o PVC, è la richiesta di storage formulata da un workload.

Nel laboratorio k3s, la StorageClass `local-path` assegna spazio sul filesystem locale del nodo.

```text
PostgreSQL Pod
      ↓
PersistentVolumeClaim
      ↓
PersistentVolume
      ↓
Storage locale del nodo k3s
```

La persistenza protegge i dati dalla semplice eliminazione o ricreazione del pod. Non protegge dalla perdita del disco, della distribuzione WSL o dell'intera macchina.

### CustomResourceDefinition

Una CustomResourceDefinition, o CRD, estende l'API Kubernetes introducendo nuovi tipi di risorsa.

CloudNativePG installa, tra le altre, la CRD:

```text
clusters.postgresql.cnpg.io
```

Dopo l'installazione della CRD, Kubernetes è in grado di accettare una risorsa:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
```

### Operator

Un operator è un controller Kubernetes che conosce il ciclo di vita di uno specifico prodotto.

CloudNativePG Operator osserva le risorse PostgreSQL di tipo `Cluster` e crea o aggiorna le risorse Kubernetes necessarie.

L'operator automatizza attività operative, ma non costituisce da solo una soluzione di alta disponibilità o Disaster Recovery. Replica, backup esterno, failover e ripristino devono essere progettati e configurati esplicitamente.

## Stato desiderato e riconciliazione

Kubernetes lavora mediante riconciliazione:

1. viene dichiarato uno stato desiderato;
2. un controller osserva lo stato reale;
3. se i due stati differiscono, il controller tenta di riallinearli.

Esempio:

```text
Stato desiderato: un pod PostgreSQL
Stato reale:       nessun pod PostgreSQL
Risultato:         il controller crea il pod
```

Per questo motivo eliminare manualmente una risorsa figlia non è sempre risolutivo: il controller proprietario può ricrearla.

## Argo CD

Argo CD applica il modello GitOps a Kubernetes. Il repository Git rappresenta lo stato desiderato dell'ambiente.

Il flusso della Case Platform è:

```text
Feature branch
      ↓ Pull Request
develop
      ↓
Argo CD
      ↓
k3s
```

Argo CD confronta i manifest presenti nel branch configurato con le risorse realmente presenti nel cluster.

### Application

Una Application Argo CD definisce:

- repository sorgente;
- branch, tag o commit;
- percorso dei manifest oppure chart Helm;
- cluster e namespace di destinazione;
- politica di sincronizzazione.

La root Application della Case Platform osserva:

```yaml
source:
  repoURL: https://github.com/landrimarina/case-platform-gitops.git
  targetRevision: develop
  path: argocd/bootstrap
```

Le Application create dalla root Application amministrano le singole componenti, per esempio:

- `cloudnative-pg-operator`;
- `case-platform-postgresql`;
- in futuro Keycloak e APISIX.

### AppProject

Un AppProject definisce i confini entro i quali un gruppo di Application può operare:

- repository sorgente consentiti;
- cluster e namespace di destinazione;
- tipi di risorsa autorizzati.

Nella Case Platform l'AppProject `case-platform` governa le applicazioni della piattaforma e dei domini.

### Stati Argo CD

| Stato | Significato |
| --- | --- |
| `Synced` | Stato Git e stato Kubernetes coincidono |
| `OutOfSync` | Esiste una differenza da applicare o correggere |
| `Healthy` | Le risorse risultano operative |
| `Progressing` | Creazione o aggiornamento ancora in corso |
| `Degraded` | Una o più risorse presentano problemi |
| `Missing` | Una risorsa prevista non esiste nel cluster |

`Synced` e `Healthy` descrivono aspetti differenti. Un'applicazione può essere `Synced` perché i manifest sono stati applicati, ma ancora `Progressing` mentre i pod si stanno avviando.

### Refresh e Sync

Il **Refresh** chiede ad Argo CD di rileggere e confrontare le sorgenti.

Il **Hard Refresh** invalida anche la cache delle sorgenti e forza una lettura più completa.

Il **Sync** applica al cluster lo stato desiderato presente nel repository.

## Helm e Kustomize

### Helm

Helm distribuisce applicazioni Kubernetes attraverso pacchetti chiamati chart.

Nella Case Platform il chart Helm ufficiale di CloudNativePG installa:

- controller dell'operator;
- CRD;
- RBAC;
- webhook;
- risorse di supporto.

Argo CD può leggere direttamente un chart Helm e applicarlo senza eseguire manualmente `helm install`.

### Kustomize

Kustomize compone e personalizza manifest YAML senza utilizzare template.

La struttura adottata separa:

- `base`: configurazione comune;
- `overlays/local`: personalizzazioni del laboratorio.

Esempio:

```text
postgresql/
├── base/
│   ├── kustomization.yaml
│   └── postgresql-cluster.yaml
└── overlays/
    └── local/
        ├── kustomization.yaml
        └── patch-resources.yaml
```

La base descrive PostgreSQL; l'overlay locale specifica StorageClass e risorse adeguate al laboratorio.

## Comandi kubectl principali

### Informazioni sul cluster

Mostra i nodi:

```bash
kubectl get nodes
```

Mostra informazioni aggiuntive, tra cui IP e versione:

```bash
kubectl get nodes -o wide
```

Mostra gli endpoint principali del cluster:

```bash
kubectl cluster-info
```

Mostra le StorageClass:

```bash
kubectl get storageclass
```

### Namespace

Elenca i namespace:

```bash
kubectl get namespaces
```

Forma abbreviata:

```bash
kubectl get ns
```

Mostra le principali risorse di un namespace:

```bash
kubectl get all -n <namespace>
```

`kubectl get all` non mostra realmente ogni tipo di risorsa: Secret, ConfigMap, PVC, PV e CRD devono essere richiesti esplicitamente.

Elimina un namespace e tutte le risorse che contiene:

```bash
kubectl delete namespace <namespace>
```

Questo comando è distruttivo e deve essere utilizzato soltanto dopo aver verificato il contenuto del namespace.

### Pod

Mostra i pod del namespace corrente:

```bash
kubectl get pods
```

Mostra i pod di uno specifico namespace:

```bash
kubectl get pods -n <namespace>
```

Mostra i pod di tutti i namespace:

```bash
kubectl get pods -A
```

Mostra nodo, IP e altre informazioni:

```bash
kubectl get pods -A -o wide
```

Segue in tempo reale le variazioni:

```bash
kubectl get pods -n <namespace> --watch
```

Il watch si interrompe con `Ctrl+C`.

Descrive un pod e mostra eventi, probe, volumi e motivi degli errori:

```bash
kubectl describe pod <pod> -n <namespace>
```

Mostra i log:

```bash
kubectl logs <pod> -n <namespace>
```

Segue i log in tempo reale:

```bash
kubectl logs -f <pod> -n <namespace>
```

Se il pod contiene più container:

```bash
kubectl logs <pod> -n <namespace> -c <container>
```

Mostra i log dell'istanza precedente di un container riavviato:

```bash
kubectl logs <pod> -n <namespace> --previous
```

### Deployment e StatefulSet

Elenca i Deployment:

```bash
kubectl get deployments -A
```

Controlla il completamento di un rollout:

```bash
kubectl rollout status deployment/<deployment> -n <namespace>
```

Elenca gli StatefulSet:

```bash
kubectl get statefulsets -A
```

Forma abbreviata:

```bash
kubectl get sts -A
```

### Service e Ingress

Elenca i Service:

```bash
kubectl get services -A
```

Forma abbreviata:

```bash
kubectl get svc -A
```

Elenca gli Ingress:

```bash
kubectl get ingress -A
```

Descrive un Service:

```bash
kubectl describe service <service> -n <namespace>
```

### Storage

Elenca i PersistentVolumeClaim:

```bash
kubectl get pvc -A
```

Elenca i PersistentVolume:

```bash
kubectl get pv
```

Descrive un PVC, utile quando rimane in stato `Pending`:

```bash
kubectl describe pvc <pvc> -n <namespace>
```

### ConfigMap e Secret

Elenca le ConfigMap:

```bash
kubectl get configmaps -n <namespace>
```

Elenca i Secret senza mostrarne i valori:

```bash
kubectl get secrets -n <namespace>
```

Mostra struttura e metadati di un Secret:

```bash
kubectl describe secret <secret> -n <namespace>
```

I valori sensibili recuperati dal cluster non devono essere copiati nei log, nella documentazione o nei repository.

### Eventi e diagnostica

Mostra gli eventi di tutti i namespace ordinati temporalmente:

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Mostra gli eventi di un namespace:

```bash
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

Mostra consumo CPU e memoria, se Metrics Server è disponibile:

```bash
kubectl top nodes
kubectl top pods -A
```

Mostra la definizione YAML corrente di una risorsa:

```bash
kubectl get <tipo> <nome> -n <namespace> -o yaml
```

### CRD e operatori

Elenca tutte le CRD:

```bash
kubectl get crd
```

Filtra le CRD CloudNativePG:

```bash
kubectl get crd | grep postgresql.cnpg.io
```

Mostra i cluster PostgreSQL:

```bash
kubectl get clusters.postgresql.cnpg.io -A
```

Mostra il cluster PostgreSQL della piattaforma:

```bash
kubectl get cluster case-platform-postgresql \
  -n case-platform-data \
  -o wide
```

Descrive il cluster e i relativi eventi:

```bash
kubectl describe cluster case-platform-postgresql \
  -n case-platform-data
```

### Argo CD

Elenca le Application:

```bash
kubectl get applications -n argocd
```

Mostra informazioni dettagliate sulla root Application:

```bash
kubectl describe application case-platform-bootstrap -n argocd
```

Forza un Hard Refresh della root Application:

```bash
kubectl annotate application case-platform-bootstrap \
  -n argocd \
  argocd.argoproj.io/refresh=hard \
  --overwrite
```

Espone temporaneamente l'interfaccia Argo CD su `localhost`:

```bash
kubectl port-forward service/argocd-server \
  -n argocd \
  8080:443
```

L'interfaccia diventa raggiungibile all'indirizzo:

```text
https://localhost:8080
```

Il port-forward rimane attivo finché il comando è in esecuzione e si interrompe con `Ctrl+C`.

### Kustomize

Genera il manifest risultante senza applicarlo:

```bash
kubectl kustomize <directory>
```

Validazione del bootstrap Argo CD:

```bash
kubectl kustomize argocd/bootstrap
```

Validazione di PostgreSQL nell'ambiente locale:

```bash
kubectl kustomize \
  applications/platform/data/postgresql/overlays/local
```

La generazione corretta conferma che Kustomize riesce a leggere e comporre i file. Non garantisce da sola che immagini, storage e workload possano avviarsi correttamente nel cluster.

## Procedura generale di diagnosi

Quando una componente non parte, procedere in questo ordine:

1. verificare lo stato dell'Application Argo CD;
2. verificare che l'Application sia `Synced`;
3. individuare namespace e pod;
4. controllare lo stato dei pod;
5. eseguire `describe` sul pod;
6. controllare gli eventi del namespace;
7. leggere i log del container;
8. verificare Service, ConfigMap, Secret e PVC;
9. risalire alla risorsa che controlla il pod.

Comandi di base:

```bash
kubectl get applications -n argocd
kubectl get pods -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
kubectl logs <pod> -n <namespace>
kubectl get svc,configmap,secret,pvc -n <namespace>
```

## Regole operative

- Le modifiche permanenti devono essere introdotte tramite Git e Argo CD.
- `kubectl` viene utilizzato principalmente per osservazione, diagnostica e verifiche.
- Evitare modifiche manuali alle risorse governate da Argo CD: il self-healing può annullarle.
- Non eliminare un pod senza avere identificato il controller proprietario.
- Non eliminare namespace, PVC o cluster database senza una verifica preventiva.
- Non salvare credenziali o valori decodificati dei Secret nei repository.
- Prima del merge, validare sempre i manifest Kustomize.
- Dopo il merge, verificare sia lo stato Argo CD sia lo stato reale dei workload.
- Nel laboratorio a nodo singolo, `Running` non equivale ad alta disponibilità.

## Componenti attualmente installati

Al momento l'ambiente comprende:

| Componente | Ruolo |
| --- | --- |
| k3s | Cluster Kubernetes locale |
| Traefik | Ingress Controller |
| Metrics Server | Metriche di base Kubernetes |
| Local Path Provisioner | Provisioning dello storage locale |
| Argo CD | Continuous Delivery GitOps |
| CloudNativePG Operator | Gestione Kubernetes-native di PostgreSQL |
| PostgreSQL | DBMS relazionale della Case Platform |

Le componenti successive previste sono Keycloak e APISIX.
