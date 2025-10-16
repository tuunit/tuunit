---
date: 2024-05-10T20:05:47+02:00
title: GKE with Gateway API + Istio
---

Previous blog post: [Gateway API + Istio + OAuth2 Proxy](../2024-04-21-gateway-api-istio-oauth2-proxy)

In my previous blog post I demonstrated how I tested and validated that Istio with the new Gateway API + OAuth2 Proxy authorization is a feasible solution for protecting applications. With a generic, application transparent method, managed from a central location. This has proven to be quite important for Platform and SRE teams, that provide internal tooling and services for developers. This way developers can do what they do best. Focus on build applications and not how to harden complex Kubernetes and API gateways.

With this in mind, how can we make this work in GKE? They offer their own [Gatway API implementation](https://cloud.google.com/blog/products/containers-kubernetes/google-kubernetes-engine-gateway-controller-is-now-ga) backend by managed LoadBalancers for handling DNS, certificates and DDoS protection through Cloud Armor. To be able to use all those amazing features, we need to stick with the GKE provided Gateway API. Side note, if you didn't know I implemented the configuration flag for activation Gateway API for GKE in the terraform provider. Have a look at my blog post "[How Google applies "magic" to Terraform](../2023-01-28-google-terraform-magic)" for more details.

But in this scenario the question is, how can we still benefit off of Istio? Yes, exactly, by using both stacked on top of each other. This way we can still use Istio for the benefit of in-cluster mTLS encryption, observability and authorization. 

So what does that mean? How did we implement this?

Let's recapitulate what I demonstrated last time. First we need to compare the rather easy local setup of kind with the necessary architecture changes for GKE.

Architecture for a local kind setup:

![](istio-gateway-oauth2-proxy.svg)

Adapted changes for GKE to be able to use all the managed offerings regarding LoadBalancing and DDoS protection:

![](gke-istio-gateway-oauth2-proxy.svg)

As you can see for the application and SSO level no much has changed but the complexity in the gateway namespace drastically increased. In simple words what needs to be done to make this work. Is a two staged Gateway approach, first the GKE Gateway and secondly the Istio Gateway. On the GKE Gateway level we define https redirects and how the TLS termination is supposed to be done. 

Now we can dive into the actual implementation details.

By creating a Gateway with the GKE implementation the Google Cloud creates a managed load balancer instance in the project your GKE is located.

```yaml
# gke-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: gke-gateway
  namespace: gateway
spec:
  addresses:
  - type: NamedAddress
    value: gke-gateway # reserved address in your Google Cloud project
  gatewayClassName: gke-l7-global-external-managed
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      kinds: 
        - kind: HTTPRoute
      namespaces:
        from: Same

  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: gke-gateway
    allowedRoutes:
      kinds: 
        - kind: HTTPRoute
      namespaces:
        from: Same # Only HTTPRoutes from the gateway namespace are allowed to attach to this gateway. This prevents circumvention of the istio-gateway
```

Here we face the first hurdle, we need to configure the TLS secret for the TLS termination by the Google Cloud Load Balancer.

```yaml
# gke-gateway-secret.yaml
kind: Secret
metadata:
  name: gke-gateway
  namespace: gateway
type: kubernetes.io/tls
data:
  tls.crt: "..."
  tls.key: "..."
```

Enforce a redirect of all http requests to https:
```yaml
# https-redirect.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: https-redirect
  namespace: gateway
spec:
  parentRefs:
  - kind: Gateway
    name: gke-gateway
    namespace: gateway
    sectionName: http
  rules:
  - filters:
    - requestRedirect:
        scheme: https
        statusCode: 302
      type: RequestRedirect
    matches:
    - path:
        type: PathPrefix
        value: /
```

For enforcing [SSL Policies](https://cloud.google.com/load-balancing/docs/ssl-policies-concepts) configured in your Google Cloud project, you can configure the managed Load Balancer behind the GKE Gateway to adopt it by defining a so called `GCPGatewayPolicy`:
```yaml
# ssl-policy.yaml
apiVersion: networking.gke.io/v1
kind: GCPGatewayPolicy
metadata:
  name: ssl-policy
  namespace: gateway
spec:
  default:
    sslPolicy:  "your-ssl-policy-name"
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: gke-gateway
    namespace: gateway
```

You should now have a new managed Load Balancer in your Google Cloud project. Traffic logs cannot be seen anywhere in GKE but instead in Cloud Logging. This is because the Gateway implementation of GKE doesn't actually deploy any workloads in your cluster but instead triggers changes to Cloud resources.

Next, we need to create the Istio gateway in the same namespace:
```yaml
# istio-gateway.yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: istio-gateway
  namespace: gateway
  annotations:
    # The istio-gateway will not be exposed as LoadBalancer type as all traffic needs to go through the gke-gateway first
    networking.istio.io/service-type: ClusterIP
    # Backend config for configuring the health check
    cloud.google.com/backend-config: '{"default": "istio-health-check"}'
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All # The istio-gateway can be used by all developers from all namespaces. The gke-gateway can only be used from within the gateway namespace and is therefore the sole responsibility of the SRE team
 ```
 
 For the Google Cloud Load Balancer to be able to work properly with the Istio Gateway, we need to create a `GCPBackendPolicy` to configure the traffic flow and availability / health checks. In our case the Istio Gateway creates a deployment and service by the name `istio-gateway` as defined above.

Unfortunately, by default the health checks for Google Cloud Load Balancers are executed against port 80 and cannot be configured through the `GCPBackendPolicy`. Another CRD is offered by the Google Cloud for this purposed called `BackendConfig`:

```yaml
# gke-health-check.yaml
---
apiVersion: networking.gke.io/v1
kind: GCPBackendPolicy
metadata:
  name: istio-gateway-istio
  namespace: gateway
spec:
  default:
    timeoutSec: 3600
  targetRef:
    group: ""
    kind: Service
    name: istio-gateway-istio
---
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: istio-health-check
  namespace: gateway
spec:
  healthCheck:
    requestPath: /healthz/ready
    port: 15021 # istio status port
    type: HTTP
```

We are done with the load balancing and routing part. Incoming requests will now go through the Google managed Load Balancer infrastructure.

* It does the TLS termination using the certificate secret provided in the cluster
* We can view logs and have insights in Google Cloud Logging
* We can attach Google Cloud Armor and more.
 
After the Google Load Balancer the traffic gets redirect into the istio-gateway deployment which will then process all existing HTTPRoutes in the different namespaces and matches the hostnames and paths accordingly.

This way we have the full power of Googles Load Balancer protection as well as the observability and security capabilities of Istio on GKE in one package.

Read more about how to integrate OAuth2 Proxy: [Adding OAuth2 Proxy to GKE with Gateway API + Istio](../2024-05-11-adding-oauth2-proxy-to-gke-istio/)

Furthermore, for Platform & SRE teams I would advice to adapt your RBAC roles accordingly and introduce two separate groups.

Developers: Get access to the usual resources like Deployments, StatefulSets, Services, Pods, SAs, ConfigMaps, Secrets and additionally the new HTTPRoutes. What they don't get access to are the `AuthorizationPolicy` resources from Istio and the `Gateway` resource.

The Platform / SRE team: Has full control over the `Gateway` and `AuthorizationPolicy`'s applied to each namespace and can enforce strict security measures or in case of a breach fully deactivate traffic to a namespace if necessary.
