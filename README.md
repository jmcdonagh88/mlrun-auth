# mlrun-auth

## Install Keycloak

1. Create a folder on the node to host the data that the keycloak DB will use - `/home/iguazio/keycloak_pvc` (path is in `pv.yaml`)
   1. The folder needs to have permissions 0777 (for now) - keycloak and the db are running with security_context etc., so this is just to avoid setting permissions properly.
2. Create `keycloak` ns on k8s, and the pv/pvc needed for the keycloak db (files are in the `keycloak` folder):

   ```sh
   kubectl create namespace keycloak
   kubectl -n keycloak apply -f pv.yaml
   kubectl -n keycloak apply -f pvc.yaml
   ```

3. In the `myvalues.yaml` file, set `ingress.hostname` to a proper host name for the cluster selected.
4. On the data node, run the Keycloak Helm chart:

   ```sh
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm -n keycloak install keycloak -f myvalues.yaml bitnami/keycloak
   ```

   Wait for the `keycloak-0` pod to be ready.
