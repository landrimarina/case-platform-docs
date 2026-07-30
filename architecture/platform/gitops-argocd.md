# GitOps e Continuous Delivery con Argo CD

**Stato:** Bootstrap iniziale completato, configurazione in evoluzione  
**Ultimo aggiornamento:** 23 luglio 2026

## 1. Scopo

Questo documento descrive il modello GitOps della Case Platform e il ruolo di Argo CD nel processo di Continuous Delivery.

Le istruzioni operative per sincronizzazione, diagnostica e rollback saranno mantenute in guide separate.

## 2. Principio GitOps

Nel modello GitOps, Git rappresenta la fonte dichiarativa dello stato desiderato del cluster:

```text
Git
→ stato desiderato

Cluster Kubernetes
→ stato effettivo

Argo CD
→ confronto e riconciliazione
```

Le modifiche al cluster devono essere introdotte modificando i manifest versionati, non intervenendo manualmente sulle risorse applicative.

## 3. Responsabilità di Argo CD

Argo CD:

- legge la configurazione dal repository GitOps;
- confronta lo stato desiderato con quello effettivo;
- applica le modifiche approvate;
- rileva le divergenze;
- ripristina lo stato dichiarato quando il self-healing è abilitato;
- elimina risorse non più dichiarate quando il pruning è abilitato;
- espone stato di sincronizzazione e salute;
- mantiene una traccia delle revisioni distribuite.

Argo CD non:

- compila il codice;
- esegue i test applicativi;
- costruisce immagini;
- sostituisce GitHub Actions;
- contiene la logica applicativa.

## 4. Separazione CI/CD

```text
Codice sorgente
→ GitHub Actions
→ test e build
→ pubblicazione artefatto
→ aggiornamento case-platform-gitops
→ Argo CD
→ Kubernetes/k3s
```

| Responsabilità | Componente |
| --- | --- |
| Build e test | GitHub Actions |
| Produzione immagini | GitHub Actions |
| Stato desiderato | `case-platform-gitops` |
| Riconciliazione | Argo CD |
| Esecuzione | Kubernetes/k3s |

## 5. Repository GitOps

Il repository:

```text
case-platform-gitops
```

contiene:

```text
case-platform-gitops/
├── applications/
│   ├── platform/
│   └── domains/
├── argocd/
│   ├── bootstrap/
│   ├── projects/
│   └── applicationsets/
├── environments/
│   ├── dev/
│   ├── test/
│   └── prod/
├── helm/
├── policies/
└── README.md
```

## 6. Installazione Argo CD

Nell’ambiente locale è stata installata la versione:

```text
Argo CD 3.4.5
```

Argo CD è eseguito nel namespace:

```text
argocd
```

Componenti verificati in stato `Running`:

- Application Controller;
- ApplicationSet Controller;
- Dex Server;
- Notifications Controller;
- Redis;
- Repository Server;
- API Server.

L’installazione iniziale è stata eseguita come bootstrap del cluster. Le evoluzioni successive devono essere governate tramite Git.

## 7. Root Application

La Root Application costituisce il punto di ingresso del modello GitOps:

```text
case-platform-bootstrap
```

Configurazione:

```text
Repository: case-platform-gitops
Branch: develop
Path: argocd/bootstrap
Namespace: argocd
```

Flusso:

```text
Root Application
→ legge argocd/bootstrap
→ crea le risorse Argo CD di piattaforma
→ abilita la gestione delle applicazioni successive
```

La Root Application viene applicata manualmente una sola volta. Dopo il bootstrap, Argo CD governa le risorse dichiarate nel repository.

## 8. AppProject

È stato definito l’AppProject:

```text
case-platform
```

L’AppProject delimita:

- repository sorgente autorizzato;
- cluster di destinazione;
- namespace consentiti;
- risorse cluster-scoped autorizzate;
- risorse namespace-scoped autorizzate.

Namespace inizialmente previsti:

```text
argocd
keycloak
case-platform
case-platform-anc
```

I permessi devono essere ampliati soltanto quando viene introdotta una nuova responsabilità.

## 9. Kustomize

La directory:

```text
argocd/bootstrap
```

utilizza Kustomize per comporre le risorse di bootstrap.

Struttura iniziale:

```text
argocd/
├── root-application.yaml
└── bootstrap/
    ├── kustomization.yaml
    └── case-platform-project.yaml
```

Prima del merge, i manifest devono essere verificati con:

```bash
kubectl kustomize argocd/bootstrap
kubectl apply --dry-run=server -f <manifest>
```

## 10. Sincronizzazione automatica

La Root Application prevede:

- sincronizzazione automatica;
- pruning;
- self-healing;
- server-side apply;
- creazione controllata dei namespace.

### Self-healing

Se una risorsa gestita viene modificata manualmente nel cluster, Argo CD rileva la divergenza e ripristina lo stato dichiarato in Git.

### Pruning

Se una risorsa viene rimossa da Git, Argo CD può rimuoverla dal cluster.

Il pruning richiede particolare attenzione: l’eliminazione dal repository può determinare l’eliminazione della risorsa Kubernetes.

## 11. Stati principali

### Stato di sincronizzazione

| Stato | Significato |
| --- | --- |
| `Synced` | Cluster allineato a Git |
| `OutOfSync` | Cluster diverso dallo stato dichiarato |
| `Unknown` | Stato non determinabile |

### Stato di salute

| Stato | Significato |
| --- | --- |
| `Healthy` | Risorse operative |
| `Progressing` | Deployment in corso |
| `Degraded` | Risorse non funzionanti |
| `Missing` | Risorsa attesa non presente |
| `Suspended` | Risorsa sospesa |

## 12. Branch e ambienti

Il modello previsto è:

```text
develop → DEV/integrazione
main    → baseline stabile/produzione
```

Le feature vengono sviluppate su:

```text
feature/*
```

e integrate tramite:

```text
feature/* → develop → main
```

La promozione verso un ambiente superiore deve avvenire tramite modifica Git e pull request.

## 13. Application di piattaforma

Argo CD governerà componenti condivisi come:

- Keycloak;
- BFF;
- frontend condiviso;
- capability backend condivise;
- observability;
- eventuale API Gateway.

Ogni componente deve avere:

- Application esplicita;
- repository e revisione definiti;
- path dichiarato;
- namespace di destinazione;
- policy di sincronizzazione;
- health check;
- strategia di aggiornamento.

## 14. Application di dominio

Ogni dominio full-stack potrà produrre più artefatti:

```text
case-platform-anc
├── MFE ANC
└── backend ANC
```

Il repository GitOps stabilisce:

- versioni delle immagini;
- configurazioni per ambiente;
- namespace;
- ingress;
- risorse;
- dipendenze infrastrutturali;
- policy di deployment.

Argo CD non legge direttamente il codice applicativo: legge la configurazione che indica quali artefatti distribuire.

## 15. Sicurezza

Principi:

- repository Git come fonte controllata;
- pull request obbligatorie;
- least privilege negli AppProject;
- nessun secret in chiaro nel repository;
- versioni e immagini esplicite;
- divieto di usare tag `latest`;
- accesso amministrativo ad Argo CD limitato;
- tracciabilità delle revisioni;
- aggiornamenti di operatori e componenti pianificati;
- separazione tra credenziali CI e credenziali CD.

Il repository GitOps pubblico non deve contenere:

- password;
- token;
- chiavi private;
- client secret;
- certificati privati;
- credenziali di database.

## 16. Keycloak come prima Application

Keycloak sarà la prima capability di piattaforma gestita tramite Argo CD.

Il percorso previsto è:

```text
case-platform-gitops/
└── applications/
    └── platform/
        └── keycloak/
```

La configurazione dovrà includere:

- versione fissata;
- namespace;
- Operator;
- persistenza;
- database;
- secret referenziati;
- esposizione HTTPS;
- realm di piattaforma;
- client BFF;
- mapping dei ruoli di dominio.

## 17. Operazioni manuali consentite

Le operazioni manuali devono essere limitate a:

- bootstrap iniziale;
- diagnostica;
- accesso controllato;
- emergenze documentate;
- operazioni non ancora automatizzate.

Ogni correzione stabile effettuata manualmente deve essere riportata in Git, altrimenti il self-healing potrà annullarla.

## 18. Verifiche descrittive

Comandi utili per verificare lo stato:

```bash
kubectl get pods -n argocd
kubectl get applications -n argocd
kubectl get appprojects -n argocd
```

La guida operativa dedicata descriverà in seguito:

- login;
- sincronizzazione manuale;
- lettura degli eventi;
- diagnosi `OutOfSync`;
- rollback;
- gestione degli errori.

## 19. Documentazione correlata

- `architecture/platform/continuous-integration-github-actions.md`
- `development/local-development-environment.md`
- future guide operative in `development/guides`

## 20. Forzare Argocd a vedere develop che è il source che abbiamo definito nel manifest
kubectl annotate application case-platform-bootstrap \
  -n argocd \
  argocd.argoproj.io/refresh=hard \
  --overwrite
## 21 aprire interfaccia argocd
  Per aprire l’interfaccia locale di Argo CD, esegui nel terminale di VS Code:
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    poi http://localhost:8080
    admin/GhtC7Efrvv19Py04