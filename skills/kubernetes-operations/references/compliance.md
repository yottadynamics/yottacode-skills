# Kubernetes in audited environments

Layered on top of `SKILL.md`. Control mappings are a starting point for a
conversation with your auditor, not a certification.

## Control mapping

| Framework | What clusters are usually asked to demonstrate |
|---|---|
| HIPAA | Namespace isolation, secrets encrypted at rest, NetworkPolicy segmentation, audit logging, TLS in transit |
| PCI-DSS | Segmentation per zone, RBAC least-privilege, image scanning, secrets rotation, access logging |
| FedRAMP | FIPS-validated crypto, hardened nodes, continuous monitoring, supply-chain attestation |
| SOC 2 | Change management via GitOps + PR review, monitoring/alerting, access control, availability |
| NIST 800-53 | AC (RBAC), AU (audit), CM (GitOps), IA (ServiceAccounts), SC (NetworkPolicy/TLS), SI (scanning) |

## Secrets at rest

Kubernetes Secrets are base64-encoded, not encrypted. Two things are needed:
encryption at rest in etcd, and a source of truth outside the cluster.

### Encryption configuration

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      # KMS v2 is the current recommendation: keys live in an external KMS,
      # rotation does not require rewriting every object, and the DEK is
      # cached rather than stored.
      - kms:
          apiVersion: v2
          name: cluster-kms
          endpoint: unix:///var/run/kmsplugin/socket.sock
          timeout: 3s
      # Fall through to identity so existing plaintext objects stay readable
      # until they are rewritten.
      - identity: {}
```

Where an external KMS is not available, `aesgcm` is preferred over `aescbc` —
upstream flags CBC mode for padding-oracle risk. Whichever provider you pick,
encryption applies only to objects written *after* it is enabled. Force a
rewrite of existing Secrets:

```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

### Verifying encryption actually took effect

Reading through the API server always returns plaintext — it decrypts on the
way out, so `kubectl get` can never show you whether encryption is on. Read the
raw value straight out of etcd:

```bash
kubectl create secret generic enc-test -n default --from-literal=probe=value

sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/enc-test | hexdump -C | head

# Encrypted: the value begins with k8s:enc:kms:v2: (or k8s:enc:aesgcm:v1:)
# Not encrypted: the literal string "probe" and "value" are visible

kubectl delete secret enc-test -n default
```

On OpenShift, encryption is driven by the API server operator rather than a
static config file:

```bash
oc patch apiserver cluster --type merge -p '{"spec":{"encryption":{"type":"aesgcm"}}}'
oc get openshiftapiserver -o jsonpath='{.items[0].status.conditions[?(@.type=="Encrypted")].message}'
```

### External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: payment-api-secrets
  namespace: payment-service
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: payment-api-secrets
    creationPolicy: Owner
    deletionPolicy: Delete
  data:
    - secretKey: db-password
      remoteRef: {key: payment-api, property: db_password}
    - secretKey: api-token
      remoteRef: {key: payment-api, property: api_token}
```

`external-secrets.io/v1` is current; `v1beta1` is deprecated. Check the CRD
version your operator serves with `kubectl get crd externalsecrets.external-secrets.io -o jsonpath='{.spec.versions[*].name}'`.

Refreshing a Secret does not restart consumers. Pods reading it via `env` keep
the old value until they restart; a `volumeMount` updates in place but the app
must re-read the file. Use Reloader or a rollout trigger if the app cannot.

## Audit policy

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived
rules:
  # Drop the high-volume, low-value reads first — rules are evaluated in
  # order and the first match wins.
  - level: None
    verbs: ["get", "list", "watch"]
    resources:
      - group: ""
        resources: ["events", "endpoints", "configmaps"]

  - level: RequestResponse
    users: ["system:anonymous"]

  # Metadata only — RequestResponse would write secret values to the log.
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]

  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]

  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]

  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["namespaces"]

  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "apps"
        resources: ["deployments", "statefulsets", "daemonsets"]

  - level: Metadata
    resources:
      - group: ""
        resources: ["serviceaccounts/token"]

  - level: Metadata
```

Never use `RequestResponse` on `secrets` — it writes the secret material into
the audit log, which is usually shipped off-cluster and retained for years.

Ship logs off the node. A node that is compromised or replaced takes its local
audit log with it.

## Admission policy

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
  annotations:
    compliance/nist-800-53: "SI-7"
spec:
  background: true
  rules:
    - name: validate-registries
      # Kyverno 1.13+ sets the action per rule; the spec-level
      # `validationFailureAction` field is deprecated.
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        failureAction: Enforce
        message: "Images must come from an approved registry"
        pattern:
          spec:
            containers:
              - image: "registry.example.com/* | ghcr.io/our-org/*"
            =(initContainers):
              - image: "registry.example.com/* | ghcr.io/our-org/*"
```

`=(initContainers)` makes the block conditional — without the anchor, every pod
without init containers fails the check.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - imageReferences: ["registry.example.com/*"]
          failureAction: Enforce
          # Kyverno mutates the image reference to its digest on admission,
          # so what runs is exactly what was verified.
          mutateDigest: true
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      <cosign public key>
                      -----END PUBLIC KEY-----
```

Roll new policies out in `Audit` mode first, review what would have been
blocked, then switch to `Enforce`. Going straight to `Enforce` on a live
cluster blocks legitimate deploys during the next incident.

Exclude the policy engine's own namespace and the control plane namespaces, or
a policy failure can prevent the cluster recovering itself.

## Runtime detection

```yaml
# Falco rules
- rule: Shell spawned in container
  desc: An interactive shell started inside a running container
  condition: >
    spawned_process and container and
    proc.name in (bash, sh, zsh, dash, csh) and
    not proc.pname in (cron, supervisord)
  output: >
    Shell in container (user=%user.name container=%container.name
    pod=%k8s.pod.name ns=%k8s.ns.name image=%container.image.repository
    cmd=%proc.cmdline)
  priority: WARNING
  tags: [container, shell]

- rule: Sensitive file read in container
  desc: A container read a path that commonly holds credentials
  condition: >
    open_read and container and
    (fd.name startswith /etc/shadow or
     fd.name startswith /etc/kubernetes/pki or
     fd.name contains id_rsa)
  output: >
    Sensitive file read (file=%fd.name user=%user.name
    container=%container.name pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: CRITICAL
  tags: [container, filesystem]
```

Tune these against your own workloads before enabling alerting. Init
containers, debug sidecars, and health-check scripts legitimately spawn shells;
an untuned rule produces enough noise that the real signal gets muted.

## Evidence checklist

- [ ] Every namespace has a default-deny NetworkPolicy and an explicit DNS allow.
- [ ] No Role or ClusterRole grants `*` verbs on `*` resources outside the control plane.
- [ ] No application pod uses the `default` ServiceAccount.
- [ ] etcd encryption verified by reading a raw key, not by reading the API.
- [ ] Audit policy captures RBAC changes, `pods/exec`, and Secret metadata; logs ship off-node.
- [ ] Images pinned by digest; signatures verified at admission.
- [ ] Backups restored into a scratch namespace within the last quarter.
- [ ] Production sync requires human approval; the approving identity is recorded.

## References

- [Kubernetes encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [Kubernetes auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kyverno policy library](https://kyverno.io/policies/)
- [External Secrets Operator](https://external-secrets.io/)
- [NIST SP 800-190 — Application Container Security](https://csrc.nist.gov/publications/detail/sp/800-190/final)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
