# Installazione Apache APISIX con Helm e Argo CD

## 1. Installazione

Apache APISIX è stato installato nel cluster Kubernetes tramite il chart Helm ufficiale, gestito in modalità GitOps da Argo CD.

### Struttura nel repository GitOps

```text
case-platform-gitops/
├── applications/platform/apisix/
│   └── values.yaml
└── argocd/bootstrap/
    ├── apisix.yaml
    ├── case-platform-project.yaml
    └── kustomization.yaml
```

### Configurazione Helm

Il file `applications/platform/apisix/values.yaml` contiene la configurazione applicata al chart `apisix/apisix` versione `2.16.0`:

```yaml
replicaCount: 1

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

service:
  type: NodePort

admin:
  enabled: true
  type: ClusterIP
  enable_admin_ui: false

externalEtcd:
  host:
    - http://case-platform-apisix-etcd.apisix.svc.cluster.local:2379
  user: ""

etcd:
  enabled: true
  replicaCount: 1
  preUpgradeJob:
    enabled: false

ingress-controller:
  enabled: true
  webhook:
    enabled: false
```

La configurazione installa una replica di APISIX, una replica di etcd e l'APISIX Ingress Controller. Il Gateway è esposto come `NodePort`; l'Admin API rimane accessibile soltanto all'interno del cluster e l'interfaccia grafica amministrativa non è abilitata.

### Application Argo CD

Il file `argocd/bootstrap/apisix.yaml` definisce l'Application Argo CD. La prima sorgente recupera il chart Helm ufficiale; la seconda recupera il `values.yaml` dal repository GitOps:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apisix
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: case-platform

  sources:
    - repoURL: https://apache.github.io/apisix-helm-chart
      chart: apisix
      targetRevision: 2.16.0
      helm:
        releaseName: case-platform-apisix
        valueFiles:
          - $values/applications/platform/apisix/values.yaml

    - repoURL: <URL_REPOSITORY_CASE_PLATFORM_GITOPS>
      targetRevision: develop
      ref: values

  destination:
    server: https://kubernetes.default.svc
    namespace: apisix

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Argo CD crea automaticamente il namespace `apisix`, installa le risorse generate dal chart e riallinea il cluster quando cambia la configurazione nel branch `develop`.

### Autorizzazioni dell'AppProject

Nel file `argocd/bootstrap/case-platform-project.yaml`, il progetto `case-platform` autorizza il repository Helm, il namespace `apisix` e le risorse cluster-wide richieste dall'installazione:

```yaml
spec:
  sourceRepos:
    - https://apache.github.io/apisix-helm-chart

  destinations:
    - server: https://kubernetes.default.svc
      namespace: apisix

  clusterResourceWhitelist:
    - group: networking.k8s.io
      kind: IngressClass
    - group: apiextensions.k8s.io
      kind: CustomResourceDefinition
```

Le voci APISIX sono aggiunte alle eventuali autorizzazioni già presenti nel progetto.

### Inclusione nel bootstrap

Il file `argocd/bootstrap/kustomization.yaml` include l'Application APISIX:

```yaml
resources:
  - apisix.yaml
```

Dopo il merge nel branch `develop`, la root Application `case-platform-bootstrap` acquisisce la nuova definizione e Argo CD sincronizza automaticamente l'Application `apisix`.

## 2. Componenti APISIX installate e relativo utilizzo

### APISIX Gateway

**Risorsa Kubernetes:** `Deployment/case-platform-apisix`

Riceve il traffico applicativo, applica le regole di instradamento e le policy configurate, quindi inoltra le richieste ai servizi di backend. Può inoltre applicare plugin per autenticazione, autorizzazione, limitazione del traffico, trasformazione delle richieste, logging e osservabilità.

| Servizio | Tipo | Utilizzo |
| --- | --- | --- |
| `case-platform-apisix-gateway` | `NodePort` | Espone il Gateway e costituisce il punto di ingresso del traffico applicativo verso APISIX. |
| `case-platform-apisix-admin` | `ClusterIP` | Espone internamente l'Admin API utilizzata per configurare rotte, upstream e plugin. |

### APISIX Ingress Controller

**Risorsa Kubernetes:** `Deployment/case-platform-apisix-ingress-controller`

Osserva le risorse dichiarative pubblicate nel cluster Kubernetes e le traduce nella configurazione utilizzata da APISIX Gateway. Consente di gestire le regole di esposizione attraverso manifest Kubernetes e di mantenerle nel flusso GitOps.

| Servizio | Tipo | Utilizzo |
| --- | --- | --- |
| `case-platform-apisix-ingress-controller` | `ClusterIP` | Espone internamente le funzionalità di servizio dell'Ingress Controller. |

### etcd

**Risorsa Kubernetes:** `StatefulSet/case-platform-apisix-etcd`

Conserva la configurazione dinamica utilizzata da APISIX, inclusi rotte, upstream, servizi, consumer e plugin. Nel laboratorio è installato con una sola replica.

| Servizio | Tipo | Utilizzo |
| --- | --- | --- |
| `case-platform-apisix-etcd` | `ClusterIP` | Fornisce ad APISIX l'endpoint interno per accedere ai dati di configurazione. |
| `case-platform-apisix-etcd-headless` | Headless Service | Fornisce identità di rete stabile e discovery DNS al pod dello StatefulSet etcd. |

### IngressClass e CRD

| Risorsa | Utilizzo |
| --- | --- |
| `IngressClass` APISIX | Identifica APISIX come controller delle risorse Ingress associate alla relativa classe. |
| CRD APISIX | Introducono le risorse personalizzate per descrivere rotte, upstream, consumer e altre configurazioni specifiche di APISIX. |
| CRD Gateway API | Consentono di descrivere gateway e regole di instradamento tramite le API Kubernetes Gateway. |
