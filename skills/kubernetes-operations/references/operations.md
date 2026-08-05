# Kubernetes cluster operations

Upgrades, backup and restore, GitOps, monitoring, and supply chain.

Version numbers below are placeholders. Always read the current available
versions from the cluster before running an upgrade — do not copy a version
string out of this file.

## Pre-upgrade check

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Version ==="
kubectl version

echo "=== Nodes ==="
kubectl get nodes -o wide

echo "=== Pods not Running/Succeeded ==="
kubectl get pods -A --field-selector='status.phase!=Running,status.phase!=Succeeded' --no-headers || true

echo "=== PodDisruptionBudgets (ALLOWED DISRUPTIONS must be >= 1) ==="
kubectl get pdb -A

echo "=== Pending PVCs ==="
kubectl get pvc -A --field-selector=status.phase=Pending --no-headers || true

echo "=== Deprecated API usage ==="
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis || echo "none reported"

echo "=== Certificate expiry (kubeadm clusters) ==="
kubeadm certs check-expiration 2>/dev/null || echo "not a kubeadm cluster"
```

`apiserver_requested_deprecated_apis` reports what is being called *now*. A
controller that reconciles hourly may not appear in a short sample — also grep
your manifests for removed API versions with `kubectl-convert` or `pluto`.

## Upgrades

### OpenShift

```bash
oc get clusterversion
oc adm upgrade                       # lists available targets

CHANNEL="stable-$(oc get clusterversion version -o jsonpath='{.status.desired.version}' | cut -d. -f1,2)"
oc adm upgrade channel "${CHANNEL}"

# Pick a target from the `oc adm upgrade` output above — never hardcode
TARGET=4.x.y
oc adm upgrade --to="${TARGET}"

watch -n 30 'oc get clusterversion; echo ---; oc get clusteroperators'

# Post-upgrade: every operator should be Available=True, Progressing=False, Degraded=False
oc get clusteroperators
```

### Managed platforms

```bash
# EKS — control plane first, then add-ons
aws eks describe-cluster --name "${CLUSTER}" --query 'cluster.version'
aws eks update-cluster-version --name "${CLUSTER}" --kubernetes-version "${TARGET}"
aws eks wait cluster-active --name "${CLUSTER}"
for addon in vpc-cni coredns kube-proxy; do
  aws eks update-addon --cluster-name "${CLUSTER}" --addon-name "$addon" --resolve-conflicts PRESERVE
done

# AKS
az aks get-upgrades -g "${RG}" -n "${CLUSTER}" -o table
az aks upgrade -g "${RG}" -n "${CLUSTER}" --kubernetes-version "${TARGET}"

# GKE — control plane, then node pools
gcloud container clusters upgrade "${CLUSTER}" --master --cluster-version "${TARGET}" --region "${REGION}"
gcloud container clusters upgrade "${CLUSTER}" --node-pool="${POOL}" --region "${REGION}"

# ARO
az aro list-upgrades -g "${RG}" -n "${CLUSTER}" -o table
az aro update -g "${RG}" -n "${CLUSTER}"

# ROSA
rosa list upgrade --cluster="${CLUSTER}"
rosa upgrade cluster --cluster="${CLUSTER}" --schedule-date="${SCHEDULE_DATE}"
```

On EKS the control plane and node groups upgrade separately, and a node group
more than one minor version behind the control plane is unsupported. Upgrade
node groups promptly after the control plane.

## Backup

### etcd snapshot

Run on a control plane node. Paths differ between distributions — these are
kubeadm defaults.

```bash
SNAPSHOT="/backup/etcd-$(date +%Y%m%d-%H%M%S).db"

sudo ETCDCTL_API=3 etcdctl snapshot save "${SNAPSHOT}" \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# A snapshot that fails `status` is not a backup — check it every time
sudo ETCDCTL_API=3 etcdctl snapshot status "${SNAPSHOT}" --write-out=table
```

A snapshot without the matching PKI directory (`/etc/kubernetes/pki`) is not a
restorable backup — the certificates are needed to bring the cluster back.
Back up both, and store them somewhere the cluster's own failure cannot take
out.

OpenShift ships its own script — use `/usr/local/bin/cluster-backup.sh` rather
than calling `etcdctl` directly.

### Velero

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: payment-daily
  namespace: velero
spec:
  includedNamespaces: [payment-service, payment-db]
  excludedResources: [events, events.events.k8s.io]
  storageLocation: default
  volumeSnapshotLocations: [default]
  ttl: 2160h
  snapshotVolumes: true
  hooks:
    resources:
      - name: db-quiesce
        includedNamespaces: [payment-db]
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: payment-db
        pre:
          # Write the dump into a directory that IS part of the backed-up
          # volume, otherwise the hook produces a file nothing captures.
          - exec:
              container: postgres
              command: ["/bin/sh", "-c", "pg_dump -U postgres -F c -f /var/lib/postgresql/data/backup/pre-velero.dump"]
              onError: Fail
              timeout: 300s
        post:
          - exec:
              container: postgres
              command: ["/bin/sh", "-c", "rm -f /var/lib/postgresql/data/backup/pre-velero.dump"]
              timeout: 60s
```

```bash
velero schedule create payment-daily \
  --schedule="0 2 * * *" \
  --include-namespaces payment-service,payment-db \
  --ttl 2160h

BACKUP=$(velero backup get -o name | head -1)
velero backup describe "${BACKUP##*/}" --details
velero restore create --from-backup "${BACKUP##*/}"
```

An untested backup is a hypothesis. Restore into a scratch namespace on a
schedule and verify the data, or you will discover the gap during an incident.

## GitOps

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-api-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://git.example.com/platform/k8s-manifests.git
    # Track a tag or commit for production, not a moving branch — `main`
    # means the deployed version changes whenever someone merges.
    targetRevision: v2.1.0
    path: overlays/production/payment-service
  destination:
    server: https://kubernetes.default.svc
    namespace: payment-service
  syncPolicy:
    # Omit `automated` entirely for manual sync. Setting it to null is not
    # the documented way to disable auto-sync.
    syncOptions:
      - CreateNamespace=false
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 3
      backoff: {duration: 5s, factor: 2, maxDuration: 3m}
  ignoreDifferences:
    # An HPA owns replicas; without this ArgoCD reports permanent drift and
    # fights the autoscaler on every sync.
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

### Kustomize layout

```text
k8s-manifests/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   ├── networkpolicy-default-deny.yaml
│   └── networkpolicy-app.yaml
└── overlays/
    ├── development/kustomization.yaml
    ├── staging/kustomization.yaml
    └── production/
        └── payment-service/
            ├── kustomization.yaml
            ├── resource-quota.yaml
            └── external-secret.yaml
```

```yaml
# overlays/production/payment-service/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: payment-service
resources:
  - ../../../base
  - resource-quota.yaml
  - external-secret.yaml
labels:
  # `commonLabels` is deprecated; `labels` with includeSelectors: false avoids
  # mutating immutable Deployment selectors on existing objects.
  - pairs:
      environment: production
    includeSelectors: false
images:
  - name: registry.example.com/payment-api
    digest: sha256:0000000000000000000000000000000000000000000000000000000000000000
replicas:
  - name: payment-api
    count: 3
patches:
  - target:
      kind: Deployment
      name: payment-api
    patch: |
      - op: add
        path: /spec/template/spec/nodeSelector
        value:
          workload-class: payment
```

Render before committing: `kustomize build overlays/production/payment-service`.

## Monitoring

```bash
# Create the admin credential first — `--set adminPassword=` puts it in shell
# history and in the process table for every user on the box.
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic grafana-admin -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 32)"

helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=90d \
  --set prometheus.prometheusSpec.replicas=2 \
  --set prometheus.prometheusSpec.resources.requests.memory=4Gi \
  --set alertmanager.alertmanagerSpec.replicas=3 \
  --set grafana.admin.existingSecret=grafana-admin \
  --set grafana.admin.userKey=admin-user \
  --set grafana.admin.passwordKey=admin-password \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size=10Gi
```

### Alerts

Every expression here uses a metric that kube-state-metrics or the named
exporter actually exposes. Before adding an alert, confirm the series exists:

```bash
kubectl -n monitoring exec -it prometheus-prometheus-0 -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/label/__name__/values' | tr ',' '\n' | grep "${METRIC}"
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: platform-alerts
  namespace: monitoring
spec:
  groups:
    - name: platform.rules
      rules:
        # kube-state-metrics does not expose securityContext fields, so
        # "is this pod running as root" cannot be alerted on from KSM.
        # Enforce it at admission (Pod Security Standards or a Kyverno policy)
        # and alert on the policy engine's own results instead.
        - alert: AdmissionPolicyFailing
          expr: sum by (policy_name, rule_name) (rate(kyverno_policy_results_total{rule_result="fail"}[10m])) > 0
          for: 10m
          labels: {severity: warning}
          annotations:
            summary: "Kyverno policy {{ $labels.policy_name }} is failing admissions"

        - alert: NamespaceWithoutNetworkPolicy
          # Requires kube-state-metrics started with networkpolicies in
          # --resources; the series is absent otherwise and this never fires.
          expr: |
            count by (namespace) (kube_namespace_labels{namespace!~"kube-.*|openshift-.*|default"})
            unless
            count by (namespace) (kube_networkpolicy_labels)
          for: 30m
          labels: {severity: warning}
          annotations:
            summary: "Namespace {{ $labels.namespace }} has no NetworkPolicy"

        - alert: VeleroBackupFailed
          expr: increase(velero_backup_failure_total[1h]) > 0
          for: 5m
          labels: {severity: critical}
          annotations:
            summary: "Velero backup failed — restore capability at risk"

        - alert: PodDisruptionBudgetBlocking
          expr: kube_poddisruptionbudget_status_expected_pods - kube_poddisruptionbudget_status_current_healthy > 0
          for: 15m
          labels: {severity: warning}
          annotations:
            summary: "PDB {{ $labels.namespace }}/{{ $labels.poddisruptionbudget }} is below its minimum"
```

## Supply chain

```bash
# Fail CI on fixable HIGH/CRITICAL findings
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  --ignore-unfixed \
  registry.example.com/payment-api:2.1.0

# SBOM
trivy image --format spdx-json -o sbom-payment-api-2.1.0.json \
  registry.example.com/payment-api:2.1.0

# Sign and verify by digest — a tag can be repointed after signing
DIGEST=$(crane digest registry.example.com/payment-api:2.1.0)
cosign sign --key cosign.key "registry.example.com/payment-api@${DIGEST}"
cosign verify --key cosign.pub "registry.example.com/payment-api@${DIGEST}"

# Cluster benchmarks
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -f job/kube-bench
kubescape scan framework nsa --exclude-namespaces 'kube-system,openshift-*'
```

Quote glob patterns passed to `--exclude-namespaces`, or the shell expands
them against your working directory first.

Signature verification only has teeth when it is enforced at admission — see
`compliance.md` for the Kyverno `verifyImages` policy.
