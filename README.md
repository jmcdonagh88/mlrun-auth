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

### Configure Keycloak

1. Create a realm - `MLRun`, add roles and users as necessary.
2. Configure an `oauth` client according to the instructions here - https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider#keycloak-oidc-auth-provider

### Retrieving token from Keycloak

Once Keycloak is configured and there's a realm available, it's possible to extract a user-token using the client-id of the oauth2 proxy. The REST call to use is this:

```sh
curl \
    -d "client_id=<client ID>" -d "client_secret=<client secret>" \
    -d "username=<username>" -d "password=<password>" \
    -d "grant_type=password" \
    https://<keycloak host>/realms/<realm name>/protocol/openid-connect/token | jq -r '.access_token'
```



#### User attributes

Any user in Keycloak can have additional attributes associated with it, which can be mapped to the auth JWT passed back to the auth client. This is done through setting fields in the 
attributes section of the user. For example, you can set an attribute `gid` and set it to any value.

Then in the client mappings, add mappings of type `User Attribute` and select the attribute you want to pass and the name of the JWT claim to assign it to. For example, the same `gid` can
be mapped to a `gid` token claim and this will be available in the auth header received by the app.

## Install oauth2-proxy

1. Install Helm chart:

   ```sh
   helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
   helm -n keycloak install oauth2 -f myvalues.yaml oauth2-proxy/oauth2-proxy
   ```

## NGINX configuration and ingress annotations

1. The auth headers returned from keycloak seem to be pretty big, which need configuration tweaks:
   1. oauth2-proxy have to be deployed with redis as session storage (it's already reflected in the values)
   2. NGINX has issues with handling the headers sent through the process, which are too big for it. To solve this, either the default NGINX configuration needs to change, or an annotation
   needs to be added to every ingress (this is already in the `ingress.yaml` file used for the demo-app). I've used 16K but that's not necessarily needed, it may be enough to use a smaller
   value:

   ```yaml
   nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
   ```

2. By default, the NGINX controller would not pass authorization headers to the application. Therefore it's critical to set the following value in the annotations for each ingress:

   ```yaml
   nginx.ingress.kubernetes.io/auth-response-headers: X-Auth-Request-User,X-Auth-Request-Email,authorization
   ```

   The important one is the `authorization` field, which will pass the authorization header containing the JWT to the app. The other fields are nice-to-haves which can identify the user/email 
   without having to dig into the JWT.
