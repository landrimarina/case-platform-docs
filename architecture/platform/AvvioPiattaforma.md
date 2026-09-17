Prerequisiti (una tantum)
k3s attivo con Keycloak, PostgreSQL, APISIX già deployati via GitOps (lo start-dev.sh fa il port-forward di Keycloak da k3s).
File secret per il BFF — non versionato. Crealo:
    echo 'BFF_OIDC_CLIENT_SECRET=<valore-del-client-secret>' > case-platform-pgcc/scripts/secrets.env
    (set-env-bff.sh lo richiede, altrimenti esce con errore).
Avvio completo (tutti i servizi)
    Dalla root del workspace: ./start-dev.sh
    Avvio dei servizi:
    Servizio	        URL	                            Debug (JDWP)
    Shell (host MFE)	http://localhost:3000	        —
    PGCC MFE	http://localhost:3001	                —
    BFF (Quarkus dev)	http://localhost:8081	        5005
    PGCC Backend (Quarkus dev)	http://localhost:8082	5006
    Keycloak (port-forward k3s)	http://localhost:8180	—
Riavvio di un singolo servizio
    ./restart-service.sh <nome>
    # nomi: shell | pgcc-mfe | bff | pgcc-backend
Stop
        ./stop-dev.sh

Log
    /home/marinalandri/workspace/.dev-logs/
Note
    Gli script fissano JAVA_HOME=/opt/java/jdk-21.0.11+10 e usano Node 24 via nvm — assicurati che siano installati con quelle versioni.
    Se k3s/Keycloak non è attivo, i backend partono ma l'autenticazione OIDC fallirà: start-dev.sh avvisa con ⚠ Keycloak non trovato in k3s.
    Punto d'ingresso applicativo: apri http://localhost:3000 (la Shell chiama il BFF su /api/session/me e federa il MFE PGCC).

Avvio console ArgoCD:
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    https://localhost:8080
    Utente: admin
    password:
    kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
