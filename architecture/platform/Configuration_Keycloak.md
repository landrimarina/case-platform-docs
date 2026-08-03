# Configuration Keycloack

## console keycloack
recupero secret
    -kubectl get secrets -n keycloak

    -kubectl get secret case-platform-keycloak-initial-admin \
  -n keycloak \
  -o jsonpath='{.data.username}' | base64 -d
    echo  
    
    kubectl get secret case-platform-keycloak-initial-admin \
  -n keycloak \
  -o jsonpath='{.data.password}' | base64 -d
echo

username,password: temp-admin/a1cc2aa1d3194fa7b902108b772b3ef3
kubectl port-forward -n keycloak \
  service/case-platform-keycloak-service 8080:8080
http://localhost:8080/admin