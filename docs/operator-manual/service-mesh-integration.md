# Istio

Argo CD can be easily integrated into a Service Mesh like [Istio](https://istio.io) but there are some configuration changes you should do.

## Network Policies

With the last versions of Argo CD, there were some security mitigations introduced which maybe block the communication to or from istio. 

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