---
date: 2024-05-11T21:31:43+02:00
title: Adding OAuth2 Proxy to GKE with Gateway API + Istio
---

This is the second part of "[GKE with Gateway API + Istio](../2024-05-10-gke-gateway-api-istio)" on how to properly utilize GKE Gateway API + Istio Gateway API.
Furthermore, it build upon the authorization setup with OAuth2 Proxy demonstrated in [Gateway API + Istio + OAuth2 Proxy](../2024-04-21-gateway-api-istio-oauth2-proxy/)

Lets take the istiod(aemon) config from that blog to configure an OAuth2 Proxy as a custom extension provider:

```yaml
# Using the istiod chart from https://istio-release.storage.googleapis.com/charts
# istiod-2.yaml
pilot:
  cni:
    enabled: true
    chained: true
  env:
    ENABLE_NATIVE_SIDECARS: true

global:
  proxy:
    resources:
      requests:
        cpu: 10m
        memory: 128Mi

meshConfig:
  defaultConfig:
    gatewayTopology:
	  # This needs to be increased to two because of the GKE Gateway and Istio Gateway
      numTrustedProxies: 2
  accessLogFile: /dev/stdout
  # This part "../2024-05-18-adding-oauth2-proxy-to-gke-istio"is new
  extensionProviders:
    - name: "oauth2-proxy"
      envoyExtAuthzHttp:
        service: "oauth2-proxy.sso.svc.cluster.local"
        port: "80"
        includeHeadersInCheck: [
            "authorization",
            "cookie",
            "from",
            "accept",
            "user-agent",
            "proxy-authorization",
            "x-forwarded-host",
            "x-forwarded-for",
            "x-auth-request-redirect",
          ] # headers sent to the oauth2-proxy in the check request.
        headersToUpstreamOnAllow: [] # headers sent to backend application when request is allowed.
        headersToDownstreamOnDeny: ["content-type", "set-cookie"] # headers sent back to the client when request is denied.
```

You will have to label all namespaces that should be using the istio service mesh with `istio-injection=enabled`. Otherwise the istio sidecar injection will not work. If you do this after the fact, remember to re-roll all your deployments, statefulsets and daemonsets. I'd recommend to apply the label when initially creating the namespaces. Either via your CI/CD, GitOps or Terraform.

Now for actually using and deploying OAuth2 Proxy we do the following:

```bash
# Official helm chart
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests

# Install in sso namespace
helm install -n sso oauth2-proxy oauth2-proxy/oauth2-proxy -f oauth2-proxy-values.yaml
```

```yaml
# oauth2-proxy-values.yaml
config:
  clientID: "xxx"
  clientSecret: "xxx"
  cookieSecret: "xxx"
  configFile: |-
    upstreams = [ "static://200" ]
    cookie_domains = [".example.com"]
    email_domains = [ "*" ]
    cookie_secure = true
    skip_provider_button = true
    silence_ping_logging = true
    reverse_proxy = true
    # oidc based
    provider = "oidc"
    oidc_issuer_url="http://sso.example.com/dex"
    # or provider specifics like
    provider = "gitlab"
    gitlab_groups="admin,developer"
```

Configure HTTPRoute for OAuth2 Proxy to listen on the base domain or subdomains. This entirely depends on your Identity providers logic for redirect URLs and your API architecture (path based vs subdomains). Some providers like Dex enforce an exact match of the configured redirect URLs. Some Identity providers like Keycloak allow for transient subdomain matching. Meaning: https://abc.example.com/oauth2/callback is virtually equivalent to https://test.example.com/oauth2/callback if you have configured https://example.com/oauth2/callback as your accepted redirect URL in Keycloak.

```yaml
# oauth2-proxy-route.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: oauth2-proxy
  namespace: sso
spec:
  hostnames:
  - example.com
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: istio-gateway
    namespace: gateway
  rules:
  - backendRefs:
    - group: ""
      kind: Service
      name: oauth2-proxy
      port: 80
      weight: 1
    matches:
    - path:
        type: PathPrefix
        value: /oauth2
```

Now you can apply arbitrary `AuthorizationPolicy`'s in your namespaces. Like so:

```yaml
# auth-policy.yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: oauth2-policy
spec:
  action: CUSTOM
  provider:
    name: oauth2-proxy
  rules:
  - when:
    - key: request.headers[X-Envoy-External-Address]
      values:
      - '*'
      notValues:
      - <ip of your google cloud nat gateway>
```

This will allow internal traffic of services and health checks that connect via public DNS entries back to the inside through the Load Balancer to skip the authorization (This should be avoided whenever possible for cost and latency reasons. Best to use in-cluster addresses but whenever necessary this is the solution). Otherwise the wildcare matcher '\*' enforces the Authorization check on all other IPs.
