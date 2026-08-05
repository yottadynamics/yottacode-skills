# Kubernetes manifest patterns

Reference manifests for `kubernetes-operations`. Placeholders use
`example.com` domains and `<...>` markers — replace before applying.

## Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payment-service
  labels:
    environment: production
    team: payments
    # Pod Security Standards. Pin the version rather than tracking `latest`,
    # so a cluster upgrade cannot tighten admission under a running workload.
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.31
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.31
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.31
  annotations:
    owner: payments-team@example.com
```

## Resource quota and limit range

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: payment-service
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    persistentvolumeclaims: "10"
    requests.storage: 100Gi
    services.nodeports: "0"
    services.loadbalancers: "2"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: payment-service
spec:
  limits:
    - type: Container
      default:        {cpu: 500m, memory: 512Mi}
      defaultRequest: {cpu: 100m, memory: 128Mi}
      max:            {cpu: "4",  memory: 8Gi}
      min:            {cpu: 50m,  memory: 64Mi}
    - type: PersistentVolumeClaim
      max: {storage: 50Gi}
      min: {storage: 1Gi}
```

A ResourceQuota on `requests.cpu` makes requests mandatory for every pod in the
namespace. Pair it with the LimitRange or existing workloads without requests
will fail admission on their next rollout.

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: payment-service
  labels:
    app.kubernetes.io/name: payment-api
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/managed-by: argocd
spec:
  replicas: 3
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 600
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: payment-api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: payment-api
    spec:
      serviceAccountName: payment-api
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: payment-api
          # Digest pinning makes the rollback target unambiguous. Replace with
          # a real digest — `docker buildx imagetools inspect <ref>`.
          image: registry.example.com/payment-api@sha256:0000000000000000000000000000000000000000000000000000000000000000
          imagePullPolicy: IfNotPresent   # a digest is immutable; Always just adds registry traffic

          ports:
            - name: https
              containerPort: 8443
              protocol: TCP

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
            # Only if the process binds a port below 1024. Prefer a high port
            # and let the Service remap it.
            # capabilities:
            #   add: ["NET_BIND_SERVICE"]

          resources:
            requests: {cpu: 250m, memory: 256Mi}
            limits:   {cpu: "1",  memory: 1Gi}

          # startupProbe covers slow boots so liveness can stay aggressive.
          # failureThreshold * periodSeconds = 150s budget here.
          startupProbe:
            httpGet: {path: /healthz, port: https, scheme: HTTPS}
            periodSeconds: 5
            failureThreshold: 30
          # Liveness must not check downstream dependencies — a slow database
          # would restart every replica and turn a degradation into an outage.
          livenessProbe:
            httpGet: {path: /healthz, port: https, scheme: HTTPS}
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet: {path: /ready, port: https, scheme: HTTPS}
            periodSeconds: 5
            failureThreshold: 3

          # readOnlyRootFilesystem needs a writable mount for every path the
          # process writes.
          volumeMounts:
            - {name: tmp,         mountPath: /tmp}
            - {name: tls-certs,   mountPath: /etc/tls,         readOnly: true}
            - {name: app-config,  mountPath: /etc/payment-api, readOnly: true}

          env:
            - {name: TLS_CERT_PATH, value: /etc/tls/tls.crt}
            - {name: TLS_KEY_PATH,  value: /etc/tls/tls.key}
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: payment-api-secrets
                  key: db-password

      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 100Mi
        - name: tls-certs
          secret:
            secretName: payment-api-tls
            defaultMode: 0440
        - name: app-config
          configMap:
            name: payment-api-config

      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app.kubernetes.io/name: payment-api
                topologyKey: kubernetes.io/hostname
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app.kubernetes.io/name: payment-api
                topologyKey: topology.kubernetes.io/zone

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: payment-api
```

### Restarting pods on ConfigMap change

Do not hand-write a checksum annotation — the mechanism depends on your
templating tool:

- **Kustomize**: `configMapGenerator` appends a content hash to the ConfigMap name, so a content change renames the object and triggers a rollout automatically.
- **Helm**: `checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}` in the pod template annotations.
- **Plain manifests**: no automatic mechanism. Use `kubectl rollout restart deploy/<name>` after applying.

```yaml
# kustomization.yaml
configMapGenerator:
  - name: payment-api-config
    files:
      - app.conf
generatorOptions:
  disableNameSuffixHash: false   # the hash is the whole point
```

## ServiceAccount and RBAC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api
  namespace: payment-service
  annotations:
    # Workload identity — pick the one matching your platform
    eks.amazonaws.com/role-arn: "arn:aws:iam::<account-id>:role/payment-api"
    # iam.gke.io/gcp-service-account: payment-api@<project>.iam.gserviceaccount.com
    # azure.workload.identity/client-id: "<client-id>"
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: payment-api
  namespace: payment-service
rules:
  # resourceNames does not authorise `list`/`watch` — a collection request has
  # no name to match. Grant `get` on known names, or drop resourceNames and
  # accept namespace-wide read.
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
    resourceNames: ["payment-api-config"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["payment-api-tls"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payment-api
  namespace: payment-service
subjects:
  - kind: ServiceAccount
    name: payment-api
    namespace: payment-service
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: payment-api
```

A workload that reads its own ConfigMaps and Secrets through the API needs
`automountServiceAccountToken: true` on the pod (or a projected token volume).
If everything arrives via `volumeMounts` and `env`, leave it `false` and delete
the Role — the kubelet fetches those, not the pod.

Audit commands:

```bash
kubectl auth can-i --list --as=system:serviceaccount:payment-service:payment-api

# ClusterRoles with wildcard verbs
kubectl get clusterroles -o json \
  | jq -r '.items[] | select(.rules[]?.verbs | index("*")) | .metadata.name'

# Pods still running as the default ServiceAccount
kubectl get pods -A -o json \
  | jq -r '.items[] | select((.spec.serviceAccountName // "default") == "default")
           | "\(.metadata.namespace)/\(.metadata.name)"'
```

## NetworkPolicy

Apply the deny and the DNS allow in the same change. NetworkPolicies are
additive — a pod may do anything permitted by any policy selecting it.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payment-service
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: payment-service
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - {protocol: UDP, port: 53}
        - {protocol: TCP, port: 53}
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-payment-api
  namespace: payment-service
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: payment-api
  policyTypes: [Ingress]
  ingress:
    - from:
        # Both selectors in ONE list item = AND (that namespace AND that pod).
        # Two separate list items would mean OR.
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx-controller
      ports:
        - {protocol: TCP, port: 8443}
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-payment-api-to-db
  namespace: payment-service
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: payment-api
  policyTypes: [Egress]
  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: payment-db
      ports:
        - {protocol: TCP, port: 5432}
```

Two things that bite:

- Egress policy is enforced on the **pod IP**, not the Service IP. Traffic to a
  ClusterIP is DNAT'd to a backend pod before policy evaluation, so select the
  backend pods, not the Service.
- Not every CNI enforces NetworkPolicy. Flannel ignores it entirely. Confirm
  with a connectivity test rather than assuming.

## OpenShift Security Context Constraints

SCCs, most to least restrictive: `restricted-v2` (default), `nonroot-v2`,
`hostnetwork-v2`, `anyuid`, `privileged`. Application workloads should stay on
`restricted-v2`.

```bash
# Which SCC admitted a pod
oc get pod ${POD} -n ${NS} -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# Anything running with elevated SCCs
oc get pods -A -o json | jq -r '
  .items[]
  | select(.metadata.annotations["openshift.io/scc"] as $s
           | $s == "privileged" or $s == "anyuid")
  | "\(.metadata.namespace)/\(.metadata.name) → \(.metadata.annotations["openshift.io/scc"])"'
```

Grant via RoleBinding rather than `oc adm policy`, so the grant is in Git:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payment-api-scc-nonroot
  namespace: payment-service
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:nonroot-v2
subjects:
  - kind: ServiceAccount
    name: payment-api
    namespace: payment-service
```

## PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-api
  namespace: payment-service
spec:
  minAvailable: 2       # with replicas: 3, allows one eviction at a time
  selector:
    matchLabels:
      app.kubernetes.io/name: payment-api
```

`minAvailable` equal to `replicas` blocks every voluntary eviction, including
node drains during a cluster upgrade. Check `kubectl get pdb -A` and confirm
`ALLOWED DISRUPTIONS` is at least 1 before starting maintenance.

## HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-api
  namespace: payment-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-api
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - {type: Percent, value: 25, periodSeconds: 60}
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - {type: Percent, value: 100, periodSeconds: 60}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: {type: Utilization, averageUtilization: 70}
```

Memory utilisation is a poor autoscaling signal for anything with a garbage
collector or a cache — usage rises and never falls, so the HPA scales up and
never back down. Use CPU, or a custom queue-depth/latency metric.

When both an HPA and GitOps manage a Deployment, tell the GitOps tool to ignore
`spec.replicas` (see `operations.md`) or the two will fight indefinitely.
