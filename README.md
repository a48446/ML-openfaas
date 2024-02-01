# ML-openfaas
## 什麼是 OpenFaas
OpenFaas 讓開發者可以用自己喜歡的程式語言將已有或新開發的功能以function as a service 的方式封裝，
讓伺服器可以RESTful API 來自由呼叫faas執行對應功能
# Helm 安裝
```
網路上很多自己找
```

## Openfaas 安裝
```
首先先加入 repo
helm repo add openfaas https://openfaas.github.io/faas-netes/
然後再
helm install ml-openfaas . --namespace openfaas --set functionNamespace=openfaas-fn --set generateBasicAuth=true
```
畫面應該是這樣子，
![image](https://hackmd.io/_uploads/HJoJjIFcT.png)
然後再輸入這個取得密碼，帳號是 admin ，
```
echo $(kubectl -n openfaas get secret basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode)
```
就可以登入囉，

## Bug
```
新版的會遇到 gateway Crash 的問題，
這時候只要把 value.yaml 中的 serviceType 改成 LoadBalancer 就可以了，完整的 value.yaml 如下。
```
```
functionNamespace: openfaas-fn  # Default namespace for functions

# See https://www.openfaas.com/support for more
openfaasPRO: false

exposeServices: true

async: false

serviceType: LoadBalancer

httpProbe: true               # Setting to true will use HTTP for readiness and liveness probe on the OpenFaaS system Pods (incompatible with Istio < 1.1.5)

rbac: true

clusterRole: false            # Set to true to have OpenFaaS administrate multiple namespaces

createCRDs: true

# create pod security policies for OpenFaaS control plane
# https://kubernetes.io/docs/concepts/policy/pod-security-policy/
psp: false
securityContext: true
basic_auth: true
generateBasicAuth: false

# image pull policy for openfaas components, can change to `IfNotPresent` in offline env
openfaasImagePullPolicy: "Always"

# openfaasPRO components, which requires openfaasPRO=true
oidcAuthPlugin:
  enabled: false
  provider: "" # Leave blank, or put "azure"
  license: "example"
  insecureTLS: false
  scopes: "openid profile email"
  jwksURL: https://example.eu.auth0.com/.well-known/jwks.json
  tokenURL: https://example.eu.auth0.com/oauth/token
  audience: https://example.eu.auth0.com/api/v2/
  authorizeURL: https://example.eu.auth0.com/authorize
  welcomePageURL: https://gw.oauth.example.com
  cookieDomain: ".oauth.example.com"
  baseHost: "http://auth.oauth.example.com"
  clientSecret: SECRET
  clientID: ID
  resources:
    requests:
      memory: "120Mi"
      cpu: "50m"
  replicas: 1
  image: openfaas/openfaas-oidc-plugin:0.3.7
  securityContext: true

# Requires openfaasPRO=true
# scale-to-zero feature
faasIdler:
  image: ghcr.io/openfaas/faas-idler-pro:0.4.2
  replicas: 1
  create: true
  inactivityDuration: 30m               # If a function is inactive for 15 minutes, it may be scaled to zero
  reconcileInterval: 2m                 # The interval between each attempt to scale functions to zero
  readOnly: true                        # When set to true, no functions are scaled to zero
  resources:
    requests:
      memory: "64Mi"

## OSS components

gateway:
  image: ghcr.io/openfaas/gateway:0.27.5
  readTimeout : "65s"
  writeTimeout : "65s"
  upstreamTimeout : "60s"  # Must be smaller than read/write_timeout
  replicas: 1
  scaleFromZero: true
  # change the port when creating multiple releases in the same baremetal cluster
  nodePort: 31112
  maxIdleConns: 1024
  maxIdleConnsPerHost: 1024
  directFunctions: false
  # Custom logs provider url. For example openfaas-loki would be
  # "http://ofloki-openfaas-loki.openfaas:9191/"
  logsProviderURL: ""
  resources:
    requests:
      memory: "120Mi"
      cpu: "50m"

basicAuthPlugin:
  image: ghcr.io/openfaas/basic-auth:0.25.5
  replicas: 1
  resources:
    requests:
      memory: "50Mi"
      cpu: "20m"

faasnetes:
  image: ghcr.io/openfaas/faas-netes:0.17.6
  readTimeout : "60s"
  writeTimeout : "60s"
  imagePullPolicy : "Always"    # Image pull policy for deployed functions
  httpProbe: true               # Setting to true will use HTTP for readiness and liveness probe on Pods (incompatible with Istio < 1.1.5)
  setNonRootUser: false
  readinessProbe:
    initialDelaySeconds: 2
    timeoutSeconds: 1           # Tuned-in to run checks early and quickly to support fast cold-start from zero replicas
    periodSeconds: 2            # Reduce to 1 for a faster cold-start, increase higher for lower-CPU usage
  livenessProbe:
    initialDelaySeconds: 2
    timeoutSeconds: 1
    periodSeconds: 2           # Reduce to 1 for a faster cold-start, increase higher for lower-CPU usage
  resources:
    requests:
      memory: "120Mi"
      cpu: "50m"

# replaces faas-netes with openfaas-operator
operator:
  image: ghcr.io/openfaas/faas-netes:0.17.6
  create: false
  # set this to false when creating multiple releases in the same cluster
  # must be true for the first one only
  createCRD: true
  resources:
    requests:
      memory: "120Mi"
      cpu: "50m"

queueWorker:
  image: ghcr.io/openfaas/faas-netes:0.17.6
  # Control HA of queue-worker
  replicas: 1
  # Control the concurrent invocations
  maxInflight: 1
  gatewayInvoke: true
  queueGroup: "faas"
  ackWait : "60s"
  resources:
    requests:
      memory: "120Mi"
      cpu: "50m"

# monitoring and auto-scaling components
# both components
prometheus:
  image: prom/prometheus:v2.11.0
  create: true
  resources:
    requests:
      memory: "512Mi"
  annotations: {}

alertmanager:
  image: prom/alertmanager:v0.18.0
  create: true
  resources:
    requests:
      memory: "25Mi"
    limits:
      memory: "50Mi"

# async provider
nats:
  channel: "faas-request"
  external:
    clusterName: ""
    enabled: false
    host: ""
    port: ""
  image: nats-streaming:0.17.0
  enableMonitoring: false
  metrics:
    # Should stay off by default because the exporter is not multi-arch (yet)
    enabled: false
    image: synadia/prometheus-nats-exporter:0.6.2
  resources:
    requests:
      memory: "120Mi"

# ingress configuration
ingress:
  enabled: false
  # Used to create Ingress record (should be used with exposeServices: false).
  hosts:
    - host: gateway.openfaas.local  # Replace with gateway.example.com if public-facing
      serviceName: gateway
      servicePort: 8080
      path: /
  annotations:
    kubernetes.io/ingress.class: nginx
  tls:
    # Secrets must be manually created in the namespace.

# ingressOperator (optional) – component to have specific FQDN and TLS for Functions
# https://github.com/openfaas-incubator/ingress-operator
ingressOperator:
  image: openfaas/ingress-operator:0.6.6
  replicas: 1
  create: false
  resources:
    requests:
      memory: "25Mi"

nodeSelector: {}

tolerations: []

affinity: {}

kubernetesDNSDomain: cluster.local

istio:
  mtls: false

gatewayExternal:
  annotations: {}

```
