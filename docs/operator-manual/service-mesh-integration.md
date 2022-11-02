# Istio

Argo CD can be easily integrated into a Service Mesh like [Istio](https://istio.io) but there are some configuration changes you should do.

The Argo CD API server should be run with TLS disabled. Edit the `argocd-server` Deployment to add the `--insecure` flag to the argocd-server container command, or simply set `server.insecure: "true"` in the `argocd-cmd-params-cm` ConfigMap [as described here](server-commands/additional-configuration-method.md).

## Namespace

To activate the istio sidecar injection, the istio-injection label must be added to the argocd namespace.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
  labels:
    istio-injection: enabled
```

## Virtual Service

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: argocd-server
spec:
  gateways:
    - ingress-gateway/istio
  hosts:
    - 'external.path.to.argocd.io'
  http:
  - match:
    - gateways:
      - ingress-gateway/istio
      port: 443
    route:
    - destination:
        host: argocd-server
        port:
          number: 443
```

## Network Policies

### allow-inbound-from-ingress-to-argocd-server

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-inbound-from-ingress-to-argocd-server
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: server
      app.kubernetes.io/name: argocd-server
      app.kubernetes.io/part-of: argocd
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              namespace: ingress-gateway
          podSelector:
            matchLabels:
              app: ingress-gateway
              istio: ingress-gateway
      ports:
        - port: 8080
  policyTypes:
    - Ingress
```

With the last versions of Argo CD, there were some security mitigations introduced which can block the communication to or from istiod. To fix that issue, you should add port 15012 to the redis egress configuration in the following Network Policies.

### [argocd-redis-network-policy](https://github.com/argoproj/argo-cd/blob/master/manifests/base/redis/argocd-redis-network-policy.yaml)

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: argocd-redis-network-policy
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: argocd-redis
  policyTypes:
  - Ingress
  - Egress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: argocd-server
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: argocd-repo-server
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: argocd-application-controller
      ports:
        - protocol: TCP
          port: 6379
  egress:
    - to:
      - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP 
        - port: 15012
          protocol: TCP      
```

### [argocd-redis-ha-proxy-network-policy](https://github.com/argoproj/argo-cd/blob/master/manifests/ha/base/redis-ha/argocd-redis-ha-proxy-network-policy.yaml)

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: argocd-redis-ha-proxy-network-policy
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: argocd-redis-ha-haproxy
  policyTypes:
  - Ingress
  - Egress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: argocd-server
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: argocd-repo-server
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: argocd-application-controller
      ports:
        - port: 6379
          protocol: TCP
        - port: 26379
          protocol: TCP
  egress:
    - to:
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: argocd-redis-ha
      ports:
        - port: 6379
          protocol: TCP
        - port: 26379
          protocol: TCP
    - to:
      - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
        - port: 15012
          protocol: TCP
```

### [argocd-redis-ha-server-network-policy](https://github.com/argoproj/argo-cd/blob/master/manifests/ha/base/redis-ha/argocd-redis-ha-server-network-policy.yaml)

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: argocd-redis-ha-server-network-policy
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: argocd-redis-ha
  policyTypes:
  - Ingress
  - Egress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: argocd-redis-ha-haproxy
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: argocd-redis-ha
      ports:
        - port: 6379
          protocol: TCP
        - port: 26379
          protocol: TCP
  egress:
    - to:
      - podSelector:
          matchLabels:
            app.kubernetes.io/name: argocd-redis-ha
      ports:
        - port: 6379
          protocol: TCP
        - port: 26379
          protocol: TCP
    - to:
      - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
        - port: 15012
          protocol: TCP
```