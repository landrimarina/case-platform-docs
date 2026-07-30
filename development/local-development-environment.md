# Ambiente locale di sviluppo della Case Platform

**Stato:** In evoluzione  
**Ultimo aggiornamento:** 23 luglio 2026

## 1. Scopo

Questo documento descrive i componenti installati e configurati per costruire l’ambiente locale di sviluppo della Case Platform.

Il documento risponde alle seguenti domande:

- su quale sistema viene eseguito l’ambiente;
- quali strumenti sono installati;
- quale responsabilità ha ciascun componente;
- quali componenti risultano già verificati;
- quali componenti devono ancora essere introdotti.

Le istruzioni operative dettagliate saranno mantenute in guide di utilizzo separate.

## 2. Architettura dell’ambiente

```text
Windows 11
└── WSL 2
    └── Ubuntu
        ├── Git e GitHub CLI
        ├── Visual Studio Code Remote WSL
        ├── OpenJDK 21 Temurin
        ├── Maven
        ├── kubectl
        ├── Helm
        └── k3s
            ├── containerd
            ├── CoreDNS
            ├── Traefik
            ├── Metrics Server
            ├── Local Path Provisioner
            └── Argo CD
```

Lo sviluppo e le operazioni Git vengono eseguiti all’interno di Ubuntu/WSL. I repository non devono essere gestiti da PowerShell attraverso percorsi `\\wsl.localhost`.

## 3. Componenti installati

| Livello | Componente | Responsabilità | Stato |
| --- | --- | --- | --- |
| Host | Windows 11 | Sistema operativo della macchina | Installato |
| Virtualizzazione | WSL 2 | Esecuzione dell’ambiente Linux integrato in Windows | Installato |
| Sistema operativo | Ubuntu | Ambiente principale di sviluppo | Installato |
| IDE | Visual Studio Code | Modifica del codice e dei manifest | Installato |
| Integrazione IDE | VS Code Remote WSL | Esecuzione di VS Code nel contesto Ubuntu | Installato |
| Versionamento | Git | Gestione locale del codice sorgente | Installato |
| Integrazione GitHub | GitHub CLI `gh` | Autenticazione e operazioni GitHub | Installato e autenticato |
| Runtime Java | OpenJDK 21 Temurin | Esecuzione dei componenti Java | Installato |
| Build Java | Maven | Build, test e dipendenze Java | Installato |
| Cluster locale | k3s | Distribuzione Kubernetes locale | Installato e attivo |
| Container runtime | containerd | Esecuzione dei container nel cluster | Incluso in k3s |
| CLI Kubernetes | kubectl | Gestione e diagnostica del cluster | Installato |
| Package manager Kubernetes | Helm | Packaging e installazione dei componenti Kubernetes | Installato |
| Continuous Delivery | Argo CD 3.4.5 | Riconciliazione GitOps del cluster | Installato e attivo |

## 4. Componenti forniti da k3s

k3s include e gestisce i seguenti componenti:

| Componente | Responsabilità |
| --- | --- |
| containerd | Container runtime |
| CoreDNS | Risoluzione DNS interna al cluster |
| Traefik | Ingress Controller |
| Metrics Server | Raccolta delle metriche Kubernetes |
| Local Path Provisioner | Provisioning dello storage persistente locale |
| Service Load Balancer | Esposizione dei servizi di tipo `LoadBalancer` |

Questi componenti non sono stati installati singolarmente.

## 5. Verifiche completate

Sono state effettuate le seguenti verifiche:

- nodo k3s in stato `Ready`;
- pod di sistema k3s attivi;
- comunicazione con il cluster tramite `kubectl`;
- disponibilità dell’IngressClass;
- disponibilità della StorageClass;
- funzionamento di Helm;
- installazione di un nginx di prova tramite Helm;
- installazione di Argo CD;
- pod Argo CD in stato `Running`;
- autenticazione GitHub tramite GitHub CLI.

## 6. Repository locali

I repository sono clonati nell’ambiente Ubuntu sotto:

```text
/home/marinalandri/workspace
```

Repository attualmente predisposti:

```text
case-platform-frontend
case-platform-backend
case-platform-bff
case-platform-contracts
case-platform-gitops
case-platform-docs
case-platform-anc
```

## 7. Strategia Git

La strategia adottata è:

```text
feature/* → develop → main
```

| Branch | Responsabilità |
| --- | --- |
| `feature/*` | Sviluppo isolato di una modifica |
| `develop` | Integrazione e validazione |
| `main` | Baseline stabile e rilasciabile |

Al completamento del bootstrap iniziale, `main` e `develop` sono stati allineati. Le evoluzioni vengono sviluppate su feature branch, integrate in `develop` e promosse su `main` dopo validazione.

## 8. Comandi di verifica

### Windows PowerShell

```powershell
wsl --version
wsl --list --verbose
```

### Ubuntu

```bash
lsb_release -a
git --version
gh --version
java -version
mvn -version
k3s --version
kubectl version --client
helm version --short
```

### Cluster

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get ingressclass
kubectl get storageclass
```

### Argo CD

```bash
kubectl get pods -n argocd
kubectl get applications -n argocd
kubectl get appprojects -n argocd
```

## 9. Strumenti presenti ma non adottati dalla piattaforma

Docker Desktop e Docker CLI risultano disponibili per altre attività e precedenti POC.

La Case Platform utilizza:

```text
k3s + containerd
```

Docker non costituisce il runtime del cluster e non è previsto come builder della piattaforma.

## 10. Componenti non ancora completati

I seguenti componenti fanno parte dell’architettura prevista, ma non devono essere considerati installati o implementati:

- Keycloak;
- Quarkus nei nuovi repository;
- Kogito;
- React, Vite e Module Federation;
- BFF applicativo;
- MariaDB per i domini;
- storage S3-compatible;
- Kafka;
- API Gateway;
- stack completo di observability.

## 11. Criteri di aggiornamento

Il documento deve essere aggiornato quando:

- viene installato un nuovo componente;
- cambia una versione;
- viene modificata la topologia locale;
- viene completata una verifica;
- un componente pianificato diventa operativo;
- uno strumento viene rimosso o sostituito.

Le password, i token e i secret non devono essere riportati nel documento.

