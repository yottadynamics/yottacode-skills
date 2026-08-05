---
name: kubernetes-operations
description: Write, review, and operate Kubernetes and OpenShift workloads — manifests, RBAC, NetworkPolicy, GitOps, upgrades, backups, and troubleshooting. Use when authoring or reviewing a Deployment or Helm chart, debugging a pod that will not start or schedule, tightening RBAC or network policy, planning a cluster upgrade or node drain, or setting up ArgoCD/Flux reconciliation.
license: MIT
metadata:
  author: yottacode
---

# Kubernetes operations

Most Kubernetes incidents trace back to a manifest that was correct in isolation and wrong in combination — a probe that never passes, a rollout that cannot schedule its surge pod, an RBAC rule that grants more than it reads. Work the relevant section, and verify against the live cluster rather than the YAML.

Use `kubectl` for Kubernetes and `oc` for OpenShift/ARO/ROSA. Detect which you are on before running anything cluster-specific:

```bash
kubectl api-resources --api-group=config.openshift.io -o name 2>/dev/null | head -1
# non-empty output means OpenShift
```

## 1. Read the cluster before changing it

```bash
kubectl version
kubectl get nodes -o wide
kubectl get pods -A --field-selector='status.phase!=Running,status.phase!=Succeeded'
kubectl get events -A --sort-by='.lastTimestamp' | tail -30
```

For a workload you are about to modify, capture the current state so you can diff against it afterwards: `kubectl get deploy/<name> -n <ns> -o yaml > /tmp/before.yaml`.

## 2. Diagnose from status, not from logs first

| Status | Meaning | First command |
|---|---|---|
| `Pending` | Not scheduled | `kubectl describe pod` → events: quota, nodeSelector, taints, topology constraints |
| `ContainerCreating` | Stuck mounting | `describe` → missing Secret/ConfigMap, PVC unbound, CSI failure |
| `CrashLoopBackOff` | Exiting repeatedly | `kubectl logs <pod> --previous` |
| `ImagePullBackOff` | Cannot pull | image name, `imagePullSecrets`, registry reachability |
| `OOMKilled` | Hit the memory limit | `kubectl get pod -o jsonpath='{.status.containerStatuses[*].lastState}'` |
| `Evicted` | Node under pressure | `kubectl describe node` → `MemoryPressure`/`DiskPressure` |
| `Terminating` | Blocked deleting | `kubectl get pod -o yaml` → `finalizers` |

```bash
kubectl describe pod ${POD} -n ${NS}
kubectl logs ${POD} -n ${NS} -c ${CONTAINER} --previous
kubectl run debug-net --rm -it --image=nicolaka/netshoot -- bash   # dig, curl, tcpdump, ss
```

`Pending` with no scheduling event usually means the scheduler found no feasible node. Check the constraint stack in section 3 before adding capacity.

## 3. Do not over-constrain scheduling

The commonest self-inflicted outage: a rollout that can never place its surge pod. These four settings interact, and each one alone is reasonable.

```yaml
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0     # needs somewhere to put the extra pod
  template:
    spec:
      affinity:
        podAntiAffinity:
          # PREFERRED, not required. With `required` + hostname topology and
          # replicas == node count, the surge pod has no eligible node and the
          # rollout hangs until progressDeadlineSeconds expires.
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app.kubernetes.io/name: payment-api
                topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          # ScheduleAnyway during a rollout; DoNotSchedule turns a temporary
          # imbalance into a stuck deploy.
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: payment-api
```

Rule of thumb: use `maxUnavailable: 0` **or** hard anti-affinity, not both. If you need both, `replicas` must exceed the number of eligible nodes.

Confirm before shipping:

```bash
kubectl rollout status deploy/${NAME} -n ${NS} --timeout=5m
```

## 4. Write the security context once, correctly

The `restricted` Pod Security Standard needs all of these; missing one gets the pod rejected at admission with a message that names only the first violation.

```yaml
spec:
  serviceAccountName: payment-api      # never `default`
  automountServiceAccountToken: false  # set true only if the app calls the API
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: payment-api
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      resources:
        requests: {cpu: 250m, memory: 256Mi}
        limits:   {cpu: "1",  memory: 1Gi}
```

`readOnlyRootFilesystem: true` requires an `emptyDir` for every path the process writes — usually at least `/tmp`. Full manifest in `references/manifests.md`.

Verify against the live pod, not the manifest:

```bash
kubectl get pod ${POD} -n ${NS} -o jsonpath='{.spec.containers[*].securityContext}' | jq
```

## 5. Scope RBAC to named resources

```yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
    resourceNames: ["payment-api-config"]
```

`resourceNames` does not authorise a plain `list` or `watch` — a collection request carries no name for RBAC to match, so the request is denied even though the rule looks permissive. (Recent clusters can authorise a `list` that carries a `metadata.name` field selector, but do not rely on it unless you have confirmed the feature is enabled.) If the app needs to list, scope the rule by namespace and accept that it sees every ConfigMap there, or have it `get` known names instead.

Audit what you actually granted:

```bash
kubectl auth can-i --list --as=system:serviceaccount:${NS}:${SA}
kubectl get clusterroles -o json | jq -r '.items[] | select(.rules[]?.verbs | index("*")) | .metadata.name'
```

## 6. Default-deny the network, then allow paths explicitly

A namespace with no NetworkPolicy is fully open to every other pod in the cluster. Order matters — apply the deny and the DNS allow together, or you will break service discovery for everything in the namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

NetworkPolicies are additive: a pod is allowed traffic permitted by *any* policy selecting it. See `references/manifests.md` for the DNS, ingress, and database allow-rules.

Verify the policy does what you think:

```bash
kubectl run np-test --rm -it --image=nicolaka/netshoot -n ${NS} -- \
  sh -c 'nc -zv -w3 payment-db 5432; nslookup kubernetes.default'
```

## 7. Drain nodes with a budget in place

```bash
kubectl cordon ${NODE}

# Note: comments go above the command. A `#` after a trailing `\` ends the
# line — the remaining flags are silently dropped and run as commands.
kubectl drain ${NODE} \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=120 \
  --timeout=600s

# maintenance happens here

kubectl uncordon ${NODE}
kubectl get pods -A -o wide --field-selector spec.nodeName=${NODE}
```

Without a PodDisruptionBudget, a drain can evict every replica of a service at once:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: payment-api
```

A PDB that can never be satisfied (`minAvailable` >= `replicas`) blocks the drain forever. Check `kubectl get pdb -A` before starting.

## 8. Upgrade only after checking what breaks

```bash
# Deprecated APIs still in use — these become hard failures after the upgrade
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

kubectl get pdb -A          # can any of these block a drain?
kubectl get pvc -A --field-selector=status.phase=Pending
```

Read the target version's release notes for removed APIs before scheduling. Upgrade commands per platform (OpenShift, EKS, AKS, GKE, ARO, ROSA) are in `references/operations.md`, along with etcd and Velero backup procedures.

## 9. Reconcile from Git

Manifests live in Git; the cluster is the output, never the source. Production sync stays manual where a human approval gate is required; dev and staging can auto-sync. ArgoCD `Application` and Kustomize overlay examples are in `references/operations.md`.

Never edit live objects with `kubectl edit` in an environment ArgoCD manages — the next reconcile reverts you, and the change leaves no trace.

## Output format for manifest reviews

```markdown
## Kubernetes review

- Verdict: ready / needs changes
- Scheduling: <constraint interactions, surge capacity, PDB>
- Security context: <PSS/SCC compliance, service account, capabilities>
- RBAC: <wildcards, resourceNames misuse, default SA>
- Network: <default-deny present, DNS egress, unlabelled paths>
- Resources: <missing requests/limits, quota fit>
- Rollout safety: <probes, PDB, image pinning>

## Required changes
- `path/to/manifest.yaml:LINE` — <what to change and why>
```

## Pitfalls

1. **`maxUnavailable: 0` plus required anti-affinity** — the rollout deadlocks with no error until the progress deadline expires.
2. **`resourceNames` with `list`/`watch`** — silently grants nothing; the request is rejected at runtime.
3. **A namespace with no NetworkPolicy** — open to every pod in the cluster.
4. **`readOnlyRootFilesystem` without writable mounts** — the app crashes on first write, usually to `/tmp`.
5. **Missing resource requests** — the scheduler treats the pod as free and packs nodes until they evict.
6. **`latest` or mutable tags** — pin to a digest so a rollback actually rolls back.
7. **Base64 in a ConfigMap is not encryption** — Secrets are base64 too. Use an external secret store and enable etcd encryption.
8. **`kubectl edit` under GitOps** — reverted on next sync, invisible in history.
9. **Liveness probes that check dependencies** — a slow database restarts every replica. Liveness = "is this process wedged"; readiness = "should it get traffic".
10. **PDB that blocks its own drain** — `minAvailable` must leave room for at least one eviction.

## References

- `references/manifests.md` — full workload, RBAC, NetworkPolicy, HPA, and quota manifests.
- `references/operations.md` — upgrades, etcd/Velero backups, GitOps, monitoring, supply chain.
- `references/compliance.md` — audit policy, secrets encryption, admission policy, control mappings.
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [NSA/CISA Kubernetes Hardening Guidance](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF)
- [Kubernetes deprecation policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)
