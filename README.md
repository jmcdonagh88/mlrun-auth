# mlrun-auth

## Install Keycloak

1. Create a folder on the node to host the data that the keycloak DB will use - `/home/iguazio/keycloak_pvc` (path is in `pv.yaml`)
   1. The folder needs to have permissions 0777 (for now) - keycloak and the db are running with security_context etc., so this is just to avoid setting permissions properly.
2. Create `mlrun` ns on k8s, and the pv/pvc needed for the keycloak db (files are in the `keycloak` folder):

   ```sh
   kubectl create namespace mlrun
   kubectl -n mlrun apply -f pv.yaml
   kubectl -n mlrun apply -f pvc.yaml
   ```

3. Create a configmap with the values for the MLRun realm. The Keycloak Helm chart is configured to automatically import the configuration in this config-map and build
   the MLRun realm automatically from it:

   ```sh
   kubectl -n mlrun create configmap mlrun-realm --from-file=mlrun_realm.json
   ```

4. In the `myvalues.yaml` file, set `ingress.hostname` to a proper host name for the cluster selected.
5. On the data node, run the Keycloak Helm chart:

   ```sh
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm -n mlrun install keycloak -f myvalues.yaml bitnami/keycloak
   ```

   Wait for the `keycloak-0` pod to be ready.

### Configure Keycloak realm

> **Note:**
> If you configured the configmap with the realm details in it as described above, then you already have an MLRun realm created in Keycloak, with a single `admin` user
> configured. These steps are only needed to add more realms or configure additional clients.

1. Create a realm - `MLRun`, add roles and users as necessary.
2. Configure an `oauth` client according to the instructions here - https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider#keycloak-oidc-auth-provider
3. The `Valid redirect URIs` value should only include the auth-proxy URL. For example: `http://auth-proxy.default-tenant.app.vmdev30.lab.iguazeng.com/*`

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

### User management UI

To allow a user to have user-management capabilities on a realm, the following is needed:

1. The user must have the `manage-users` role from the `realm-management` client for the specific realm. This can be done by modifying the Role mappings for that user
2. To manage users, go to `https://<keycloak host>/admin/{realm-name}/console` and login with the user that you granted permissions to

## Install oauth2-proxy

1. Install Helm chart:

   ```sh
   helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
   helm -n mlrun install oauth2 -f myvalues.yaml oauth2-proxy/oauth2-proxy
   ```

### Configuring Helm values (and their meaning)

The values included in the `myvalues.yaml` file contain the following important configurations:

1. `clientID` - this must be the same client ID as the one configured in Keycloak for the oauth2 client
2. `clientSecret` - should be the secret created in Keycloak for the client
3. `cookieSecret` - this is just a randomized value used to sign the cookie. Generating it is demonstrated here: https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview/#generating-a-cookie-secret
4. `configFile` - this contains a bunch of configurations which are critical for our configuration. The following fields are not trivial:
   1. `oidc_issuer_url` - this should contain the name of the realm in it, the format is: `https://<keycloak host>/realms/<realm name>`
   2. `whitelist_domains` - this tells the oauth2 proxy what domains can be included in the `rd` parameter passed to the login flow. It's critical to set this, since by default it only contains the original host that issued the command. However, if we want oauth2 to have an ingress with separate hostname than the services that are authenticated, it won't work out-of-the-box. Note that the domains listed there should only contain the suffix of the domain name that is shared across services. For example, if the oauth proxy is `auth-proxy.some.domain.com` and MLRun is `mlrun.some.domain.com` then this value should be `.some.domain.com`
   3. `cookie_domains` - this controls the domain that the browser cookie generated by oauth2 proxy is valid for. By default the cookie will apply to the specific hostname that is used. Since we want to enable SSO, i.e. login for one service should work for the rest of the services in the domain, then this should be set to the domain name. In general, this value should be the same as the `whitelist_domains` value

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

## Running Python demo app

In the `python_demo_app` directory, run the following:

```sh
kubectl -n mlrun create cm python-code --from-file=web_server.py
kubectl -n mlrun apply -f pod.yaml
kubectl -n mlrun apply -f service.yaml
kubectl -n mlrun apply -f ingress.yaml
```
