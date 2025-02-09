# Walkthrough

This document covers a complete journey: Traefik Proxy => Traefik Hub API Gateway => Traefik Hub API Management

## Deploy Kubernetes

For this tutorial, we deploy Traefik Hub API Gateway on a [k3d](https://k3d.io/) cluster. It's possible to use alternatives such as [kind](https://kind.sigs.k8s.io), cloud providers, and others.

First, clone this GitHub repository:

```shell
git clone https://github.com/traefik/hub.git
cd hub
```

### Using k3d

```shell
k3d cluster create traefik-hub --port 80:80@loadbalancer --port 443:443@loadbalancer --port 8000:8000@loadbalancer --k3s-arg "--disable=traefik@server:0"
```

### Using Kind

kind requires some configuration to use an IngressController on localhost. See the following example:

<details>

<summary>Create the cluster</summary>

Ports need to be mapped for HTTP and HTTPS for kind with this config:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: traefik-hub
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 80
    protocol: TCP
  - containerPort: 30001
    hostPort: 443
    protocol: TCP
```

```shell
kind create cluster --config=src/kind/config.yaml
kubectl cluster-info
kubectl wait --for=condition=ready nodes traefik-hub-control-plane
```

And add a load balancer (LB) to it:

```shell
kubectl apply -f src/kind/metallb-native.yaml
kubectl wait --namespace metallb-system --for=condition=ready pod --selector=app=metallb --timeout=90s
kubectl apply -f src/kind/metallb-config.yaml
```

</details>

## Step 1: Deploy an API with Traefik Proxy

First, we will install Traefik Proxy with Helm:

```shell
# Add the Helm repository
helm repo add --force-update traefik https://traefik.github.io/charts
# Create a namespace
kubectl create namespace traefik
# Install the Helm chart
helm install traefik -n traefik --wait \
  --set ingressClass.enabled=false \
  --set ingressRoute.dashboard.matchRule='Host(`dashboard.docker.localhost`)' \
  --set ingressRoute.dashboard.entryPoints={web} \
  --set ports.web.nodePort=30000 \
  --set ports.websecure.nodePort=30001 \
   traefik/traefik
```

Once it's installed, we can access the local dashboard: http://dashboard.docker.localhost/

![Local Traefik Proxy Dashboard](./src/images/dashboard.png)

Without Traefik Hub, an API can be deployed with an `Ingress`, an `IngressRoute` or a `HTTPRoute`.

This tutorial implements APIs using a JSON server in Go; the source code is [here](../../src/api-server/).

Let's deploy a [weather app](../../src/manifests/weather-app.yaml) exposing an API.

```shell
kubectl apply -f src/manifests/weather-app.yaml
```

It should create the public app:

```shell
namespace/apps created
configmap/weather-data created
deployment.apps/weather-app created
service/weather-app created
```

It can be exposed with an `IngressRoute`:

```yaml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: weather-api
  namespace: apps
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`api.docker.localhost`) && PathPrefix(`/weather`)
    kind: Rule
    services:
    - name: weather-app
      port: 3000
```

```shell
kubectl apply -f src/manifests/weather-app-ingressroute.yaml
```

```shell
ingressroute.traefik.io/weather-api created
```

This API can be accessed using curl:

```shell
curl http://api.docker.localhost/weather
```

```json
{
  "public": [
    { "id": 1, "city": "GopherCity", "weather": "Moderate rain" },
    { "id": 2, "city": "City of Gophers", "weather": "Sunny" },
    { "id": 3, "city": "GopherRocks", "weather": "Cloudy" }
  ]
}
```

With Traefik Proxy, we can secure the access to this API using the Basic Authentication. To create an encoded user:password pair, the following command can be used: `htpasswd -nb user password | openssl base64`

So let's do it:

```shell
htpasswd -nb foo bar | openssl base64
```

```shell
Zm9vOiRhcHIxJDJHR0RyLjJPJDdUVXJlOEt6anQ1WFFOUGRoby5CQjEKCg==
```

```diff
--- src/manifests/weather-app-ingressroute.yaml
+++ src/manifests/weather-app-basic-auth.yaml
@@ -1,4 +1,24 @@
 ---
+apiVersion: v1
+kind: Secret
+metadata:
+  name: basic-auth
+  namespace: apps
+data:
+  users: |
+    Zm9vOiRhcHIxJDJHR0RyLjJPJDdUVXJlOEt6anQ1WFFOUGRoby5CQjEKCg==
+
+---
+apiVersion: traefik.io/v1alpha1
+kind: Middleware
+metadata:
+  name: basic-auth
+  namespace: apps
+spec:
+  basicAuth:
+    secret: basic-auth
+
+---
 apiVersion: traefik.io/v1alpha1
 kind: IngressRoute
 metadata:
@@ -13,3 +33,5 @@
     services:
     - name: weather-app
       port: 3000
+    middlewares:
+    - name: basic-auth
```

Let's apply it:

```shell
kubectl apply -f src/manifests/weather-app-basic-auth.yaml
```

```shell
secret/basic-auth created
middleware.traefik.io/basic-auth created
ingressroute.traefik.io/weather-api configured
```

And now, we can confirm it's secured using BASIC Authentication :

```shell
# This call is not authorized => 401
curl -I http://api.docker.localhost/weather
# This call is allowed => 200
curl -I -u foo:bar http://api.docker.localhost/weather
```

[Basic Authentication](https://datatracker.ietf.org/doc/html/rfc7617) worked and was widely used in the early days of the web. However, it also has a security risk: credentials can be visible to any observer when using HTTP. It uses hard-coded credentials, potentially giving more authorization than required for a specific use case.

Nowadays, those issues are addressed when using [JSON Web Tokens (JWT)](https://datatracker.ietf.org/doc/html/rfc7519). A JWT can be cryptographically verified, detach authentication from user credentials, and has an issue and expiration date. JWT can be used with Traefik Hub API Gateway, so let's upgrade our setup to Traefik Hub

## Step 2: Upgrade Traefik Proxy to Traefik Hub API Gateway

Log in to the [Traefik Hub Online Dashboard](https://hub.traefik.io), open the page to [generate a new Hub API Gateway](https://hub.traefik.io/gateways/new).

**:warning: Do not install the Hub API Gateway, but copy the token.**

Now, open a terminal and run these commands to create the secret for Traefik Hub.

```shell
export TRAEFIK_HUB_TOKEN=
```

```shell
kubectl create secret generic license --namespace traefik --from-literal=token=$TRAEFIK_HUB_TOKEN
```

Then, upgrade Traefik Proxy to Traefik Hub using the same Helm chart:

```shell
helm upgrade traefik -n traefik --wait \
  --reuse-values \
  --set hub.token=license \
  --set image.registry=ghcr.io \
  --set image.repository=traefik/traefik-hub \
  --set image.tag=v3.0.0 \
   traefik/traefik
```

Traefik Hub is 100% compatible with Traefik Proxy v3.

The dashboard is reachable (http://dashboard.docker.localhost/), which confirms that Traefik Hub API Gateway is successfully deployed.

![Local Traefik Hub Dashboard](./src/images/hub-dashboard.png)

And also confirm _Basic Auth_ is still here:

```shell
# This call is not authorized => 401
curl -I http://api.docker.localhost/weather
# This call is allowed => 200
curl -I -u foo:bar http://api.docker.localhost/weather
```

Let's secure the weather API with an API Key.

With Traefik Hub, we can use API Key as a middleware. First, we'll need to generate hash of our password. It can be done with `htpasswd` :

```shell
htpasswd -nbs "" "Let's use API Key with Traefik Hub" | cut -c 2-
{SHA}dhiZGvSW60OMQ+J6hPEyJ+jfUoU=
```

```shell
{SHA}dhiZGvSW60OMQ+J6hPEyJ+jfUoU=
```

We can now put this password in the API Key middleware:

```diff
diff -Nau src/manifests/weather-app-ingressroute.yaml src/manifests/weather-app-apikey.yaml
--- src/manifests/weather-app-ingressroute.yaml
+++ src/manifests/weather-app-apikey.yaml
@@ -1,4 +1,24 @@
 ---
+apiVersion: v1
+kind: Secret
+metadata:
+  name: apikey-auth
+  namespace: apps
+stringData:
+  secretKey: "{SHA}dhiZGvSW60OMQ+J6hPEyJ+jfUoU="
+
+---
+apiVersion: traefik.io/v1alpha1
+kind: Middleware
+metadata:
+  name: apikey-auth
+  namespace: apps
+spec:
+  plugin:
+    apikey:
+      secretValues:
+        - urn:k8s:secret:apikey-auth:secretKey
+
+---
 apiVersion: traefik.io/v1alpha1
 kind: IngressRoute
 metadata:
@@ -13,3 +33,5 @@
     services:
     - name: weather-app
       port: 3000
+    middlewares:
+    - name: apikey-auth
```

Let's apply it:

```shell
kubectl apply -f src/manifests/weather-app-apikey.yaml
```

```shell
secret/apikey-auth created
middleware.traefik.io/apikey-auth created
ingressroute.traefik.io/weather-api configured
```

And test it:

```shell
# This call is not authorized => 401
curl -I http://api.docker.localhost/weather
# Let's set the token
export API_KEY=$(echo -n "Let's use API Key with Traefik Hub" | base64)
# This call with the token is allowed => 200
curl -I -H "Authorization: Bearer $API_KEY" http://api.docker.localhost/weather
```

The API is now secured.

It's possible to handle users with an Identity Provider, but what if we want to cover _internal_ and _external_ use cases? To protect API on HTTP _verb_ level? Or to test a new version with part of the production traffic?

We'll need Traefik Hub with API Management!

## Step 3: Manage an API with Traefik Hub API Management

First, we enable API Management on Traefik Traefik Hub using the same Helm chart:

```shell
helm upgrade traefik -n traefik --wait \
  --reuse-values \
  --set hub.apimanagement.enabled=true \
   traefik/traefik
```

Traefik Hub API Management is 100% compatible with Traefik Proxy v3 and Traefik Hub API Gateway.

The dashboard is reachable (http://dashboard.docker.localhost/), which confirms that Traefik Hub API Gateway is successfuly deployed.

![Local Traefik Hub Dashboard](./src/images/hub-dashboard.png)

And also confirm that the API is still secured using an API Key:

```shell
# This call is not authorized => 401
curl -I http://api.docker.localhost/weather
# Let's set the token
export API_KEY=$(echo -n "Let's use API Key with Traefik Hub" | base64)
# This call with the token is allowed => 200
curl -I -H "Authorization: Bearer $API_KEY" http://api.docker.localhost/weather
```

Now, let's try to manage it with Traefik Hub using `API` and `APIAccess` resources:

```yaml
---
apiVersion: hub.traefik.io/v1alpha1
kind: API
metadata:
  name: weather-api
  namespace: apps
spec: {}

---
apiVersion: hub.traefik.io/v1alpha1
kind: APIAccess
metadata:
  name: weather-api
  namespace: apps
spec:
  apis:
    - name: weather-api
  everyone: true
```

We'll need to reference this API in the `IngressRoute` with an annotation:

```yaml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: weather-api
  namespace: apps
  annotations:
    hub.traefik.io/api: weather-api # <=== Link to the API using its name
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`api.docker.localhost`) && PathPrefix(`/weather`)
    kind: Rule
    services:
    - name: weather-app
      port: 3000
```

:information_source: We've also removed the API Key authentication middleware, as we'll use Traefik Hub's built-in identity provider for user and credential management. The API is still secured, as we'll see it shortly.

Let's apply it:

```shell
kubectl apply -f api-management/1-getting-started/manifests/api.yaml
```

It will create `API`, `APIAccess` and link `IngressRoute` to this API.

```shell
api.hub.traefik.io/weather-api created
apiaccess.hub.traefik.io/weather-api created
ingressroute.traefik.io/weather-api configured
```

Now, we can confirm this API is not publicly exposed:

```shell
curl -i http://api.docker.localhost/weather
```

It returns the expected 401 Unauthorized HTTP code:

```shell
HTTP/1.1 401 Unauthorized
Date: Mon, 06 May 2024 12:09:56 GMT
Content-Length: 0
```

## Step 4: Create a user for this API

Users are created in the [Traefik Hub Online Dashboard](https://hub.traefik.io/users):

![Create user admin](./api-management/1-getting-started/images/create-user-admin.png)

## Step 5: Deploy the API Portal

The user created previously will connect to an API Portal to generate an API key, so let's deploy the API Portal!

```yaml
---
apiVersion: hub.traefik.io/v1alpha1
kind: APIPortal
metadata:
  name: apiportal
  namespace: traefik
spec:
  title: API Portal
  description: "Developer Portal"
  trustedUrls:
    - api.docker.localhost

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apiportal
  namespace: traefik
  annotations:
    # This annotation link this Ingress to the API Portal using <name>@<namespace> format.
    hub.traefik.io/api-portal: apiportal@apps
spec:
  rules:
  - host: api.docker.localhost
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: apiportal
              port:
                number: 9903
```

:information_source: This API Portal is routed with the internal _ClusterIP_ `Service` named apiportal.

```shell
kubectl apply -f api-management/1-getting-started/manifests/api-portal.yaml
sleep 30
```

```shell
apiportal.hub.traefik.io/apiportal created
ingressroute.traefik.io/apiportal created
```

The API Portal should be reachable on http://api.docker.localhost

We log in with the admin user.

![API Portal Log in](./api-management/1-getting-started/images/api-portal-login.png)

And create a token for this user:

![API Portal Create Token](./api-management/1-getting-started/images/api-portal-create-token.png)

```shell
export ADMIN_TOKEN="XXX"
```

Request the API with this token: :tada:

```shell
curl -H "Authorization: Bearer $ADMIN_TOKEN" http://api.docker.localhost/weather
```

```json
{
  "public": [
    { "id": 1, "city": "GopherCity", "weather": "Moderate rain" },
    { "id": 2, "city": "City of Gophers", "weather": "Sunny" },
    { "id": 3, "city": "GopherRocks", "weather": "Cloudy" }
  ]
}
```

:information_source: If it fails with 401, just wait one minute and try again. The token needs to be sync before it can be accepted by Traefik Hub.

We can see the API available in the `apps` namespace in the portal. This first API does not come with an OpenAPI specification (OAS):

![API Portal without OAS](./api-management/1-getting-started/images/api-portal-without-oas.png)

Although not setting an OAS for an API is possible, it severely hurts getting started with API consumption. Let's see what features are unlocked if we set one. Let's deploy a [forecast app](https://github.com/traefik/hub-preview/blob/main/src/manifests/weather-app-forecast.yaml) with an OpenAPI specification:

```shell
kubectl apply -f src/manifests/weather-app-forecast.yaml
```

This time, we will specify how to get the OAS in the API _CRD_:

```yaml
---
apiVersion: hub.traefik.io/v1alpha1
kind: API
metadata:
  name: weather-api-forecast
  namespace: apps
spec:
  openApiSpec:
    path: /openapi.yaml
    override:
      servers:
        - url: http://api.docker.localhost
```

The other resources are built on the same model, as we can see in [the complete file](https://github.com/traefik/hub-preview/blob/main/api-management/1-getting-started/manifests/forecast.yaml). Let's apply it:

```shell
kubectl apply -f api-management/1-getting-started/manifests/forecast.yaml
```

```shell
api.hub.traefik.io/weather-api-forecast created
apiaccess.hub.traefik.io/weather-api-forecast created
ingressroute.traefik.io/weather-api-forecast created
```

And that's it! This time, we have documentation built from the OpenAPI specification, and we can also interactively try the API with the Try Out functionality.

![API Portal With OAS](./api-management/1-getting-started/images/api-portal-with-oas.png)

