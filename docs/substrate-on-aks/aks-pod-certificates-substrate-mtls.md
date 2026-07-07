# AKS pod certificates and Substrate mTLS

> Date: 2026-06-11

## 1. General Kubernetes use case

`PodCertificateRequest` and `ClusterTrustBundle` are complementary Kubernetes certificate/trust APIs for workload mTLS.

| Resource / projection | Problem solved | What a workload gets |
| --- | --- | --- |
| `podCertificate` projected volume / `PodCertificateRequest` | Per-Pod leaf certificate issuance and private-key projection. | A short-lived key + certificate chain, typically mounted as a credential bundle file. |
| `clusterTrustBundle` projected volume / `ClusterTrustBundle` | Cluster-scoped X.509 trust-anchor distribution. | A CA/trust bundle file used to verify peer certificates. |

```
podCertificate / PodCertificateRequest  → "give this Pod its own cert + key"
ClusterTrustBundle                       → "give this Pod the CA roots it should trust"
```

Together, they provide a Kubernetes-native way to supply workloads with both sides of mTLS: local identity and remote trust. _[verified]_

### Default leaf certificate lifecycle

- If `podCertificate.maxExpirationSeconds` is omitted, Kubernetes defaults it to `86400` seconds, i.e. 24 hours. _[verified]_
- Substrate's current signers issue certificates for `min(24h, requested maxExpirationSeconds)`. _[verified]_
- Substrate sets `beginRefreshAt = notAfter - 30m`, so kubelet should start trying to refresh about 30 minutes before expiry. Kubernetes treats this as a hint. _[verified]_
- Substrate local CA roots are generated with 365-day validity in `internal/localca.GenerateED25519CA`. There is no automatic CA-root rotation in the current implementation. _[verified]_

## 2. How Substrate uses these resources for end-to-end mTLS

Current Substrate has two certificate signer domains.

| Signer | Identity shape | Role |
| --- | --- | --- |
| `servicedns.podcert.ate.dev/identity` | DNS SANs such as `api.ate-system.svc` or `valkey-cluster-service.ate-system.svc` | Service/server TLS identity and Valkey mTLS. |
| `podidentity.podcert.ate.dev/identity` | SPIFFE-like URI: `spiffe://cluster.local/ns/<namespace>/sa/<service-account>` | Pod/workload identity, currently used as a trust domain for optional client-cert verification and session identity work. |

### `servicedns` signer usage

The `servicednssigner` lists Services in the requesting Pod's namespace, finds Services selecting the Pod, and puts names like `<service>.<namespace>.svc` into the certificate DNS SANs. _[verified]_

It currently feeds these call sites:

- **`ate-api-server` gRPC server TLS:** `--grpc-server-cred-bundle=/run/servicedns.podcert.ate.dev/credential-bundle.pem`.
- **`atenet-router` / Envoy HTTPS cert:** `--envoy-cert-path=/run/servicedns.podcert.ate.dev/credential-bundle.pem`.
- **Valkey server TLS:** `tls-cert-file` and `tls-key-file` both point at `/run/servicedns.podcert.ate.dev/credential-bundle.pem`.
- **Valkey client auth:** `tls-auth-clients yes`; clients such as the init job use the same credential bundle as `--cert` and `--key`.
- **`ate-api-server` → Valkey:** code supports `--redis-client-cert` / `ATE_API_REDIS_CLIENT_CERT`, parsed as the same credential-bundle format.

The `servicedns` leaf certs are dual-use: the signer sets both `ExtKeyUsageClientAuth` and `ExtKeyUsageServerAuth`. This is convenient, but production designs may want separate server and client identities. _[verified]_

### `podidentity` signer usage

The `podidentitysigner` creates SPIFFE-like URI identities based on namespace and ServiceAccount. _[verified]_

Current trust use:

- `ate-api-server` mounts `--workerpool-ca-certs=/run/workerpool-ca-certs/trust-bundle.pem`.
- In the base manifests, that file comes from a `clusterTrustBundle` projection selecting `podidentity.podcert.ate.dev/identity`.
- `ate-api-server` uses this CA as `ClientCAs` with `tls.VerifyClientCertIfGiven`.
- `SessionIdentity.MintCert` requires a peer client certificate; source still has TODOs around verifying pod cert ↔ session mapping.

### Credential bundle shape

Substrate's `internal/credbundle` expects one file containing:

```
PRIVATE KEY
CERTIFICATE leaf
CERTIFICATE intermediate/root...
```

That format matches Kubernetes `credentialBundlePath`. Any replacement that emits separate `tls.key`, `tls.crt`, and `ca.crt` files either needs an adapter/init step or code/config changes. _[verified]_

### Reload behavior observed in current source

| Consumer | Cert/trust file | Rotation pickup |
| --- | --- | --- |
| `ate-api-server` server TLS | `grpc-server-cred-bundle` | Good: `credbundle.Loader` parses the file on each handshake. _[verified]_ |
| `ate-api-server` Redis client TLS | `redis-client-cert`, `redis-ca-certs` | Loaded once into a `tls.Config`; reload would require restart or code change. _[verified]_ |
| `ate-api-server` workerpool client CA | `workerpool-ca-certs` | Loaded once; code has TODO to periodically reload for rotations. _[verified]_ |
| `atenet-router` / Envoy | `envoy-cert-path` | Needs verification for this exact xDS/file config. _[unknown]_ |
| Valkey | server cert/key and client CA | Needs Valkey-specific reload/restart verification. _[unknown]_ |

## 3. Current AKS blocker and static certificate workaround

The tested AKS cluster did not expose the Kubernetes APIs needed by the base manifests:

- `ClusterTrustBundle` / `clusterTrustBundle` projection.
- `PodCertificateRequest` / backing `podCertificate` projected volume support.

Because of that, the AKS overlay switched to static dev Secrets:

| Secret | Replaces | Purpose |
| --- | --- | --- |
| `ate-system/servicedns-credential-bundle` | `podCertificate` projected service-DNS credential bundle | Server/client TLS bundle used by `ate-api-server`, router/Envoy, Valkey, and the Valkey init job. |
| `ate-system/workerpool-ca-certs` | `ClusterTrustBundle` projection for `podidentity` | CA bundle used by `ate-api-server` to verify optional client certificates. |
| `ate-system/valkey-ca-certs` | CA extraction from the service-DNS CA pool | CA file used by Valkey and Valkey clients for TLS verification. |

Static workaround lifecycle:

- `create_servicedns_credential_bundle_secret` signs the leaf cert with `openssl x509 ... -days 30`. _[verified]_
- The generated local CA roots are valid for 365 days. _[verified]_
- There is no automatic rotation for the static AKS Secrets. Re-running install helpers can recreate them, but that is not a production lifecycle. _[verified]_

Conclusion: the static Secret path is good enough for AKS development bootstrap, but not sufficient as a production identity/certificate lifecycle.

## 4. Potential workarounds

### cert-manager CSI driver + trust-manager

- **Fit:** Best near-term AKS path for service/Valkey TLS.
- **Pros:** Mounted cert/key files, rotation, CA bundle distribution, Kubernetes-native operator model.
- **Risks:** Need to map SANs and file shape; reload/restart behavior still must be solved.

### SPIRE/SPIFFE

- **Fit:** Best long-term workload identity path.
- **Pros:** Strong workload attestation, X.509-SVIDs, trust bundles, SPIFFE identities.
- **Risks:** Bigger platform integration; less direct fit for service-DNS certificates and Valkey's current file contract.

### Azure Key Vault + Secrets Store CSI Driver

- **Fit:** Azure-native component cert mounting.
- **Pros:** AKS-supported, Key Vault lifecycle, autorotation of mounted content/synced Secrets.
- **Risks:** Weaker per-Pod identity; more "this workload can fetch this cert" than "kubelet issued this Pod-bound cert."

### Istio / service mesh mTLS

- **Fit:** Platform service-to-service mTLS.
- **Pros:** Managed mTLS and policy between mesh workloads.
- **Risks:** Not drop-in for components expecting cert files; Valkey and SessionIdentity still need separate handling.

### Custom rotating Secret controller

- **Fit:** Bridge preserving current semantics.
- **Pros:** Can reuse `servicednssigner` / `podidentitysigner` ideas and exact bundle format.
- **Risks:** High ownership burden: rotation, key security, CA rollover, reload orchestration.

### Wait for native Kubernetes APIs on AKS

- **Fit:** Cleanest long-term match to current base manifests.
- **Pros:** Minimal redesign once both APIs are available.
- **Risks:** Blocked by AKS API availability, especially `PodCertificateRequest`.

### Refined recommendation

Split the problem into two tracks:

1. **Near-term service/Valkey TLS on AKS:** use cert-manager CSI driver + trust-manager, assuming a spike confirms SANs, file shape, and reload strategy.
2. **Longer-term workload/session identity:** evaluate SPIRE/SPIFFE if `podidentity` and `SessionIdentity` become production-critical.

_[Bayesian rule 2: strongest counter]_ The cleanest architecture is still native `podCertificate` + `ClusterTrustBundle` once AKS supports both. cert-manager/SPIRE are substitutes, not identical semantics.

## 5. Stability timeline for native Kubernetes resources

| Feature | KEP | Lifecycle | Planning read |
| --- | --- | --- | --- |
| `ClusterTrustBundle` object | KEP-3257 | Alpha v1.27; beta v1.33; upstream stable target v1.37. | Closer to GA. |
| `clusterTrustBundle` projection | KEP-3257 | Alpha v1.29; beta v1.33; upstream stable target v1.37. | Closer to GA. |
| `PodCertificateRequest` / `podCertificate` projection | KEP-4317 | Alpha v1.34; beta v1.35; active beta work in v1.36; no stable milestone recorded in KEP YAML. | The real timeline blocker. |

_[verified]_ KEP-3257 metadata says `stage: stable`, `latest-milestone: v1.37`, and `stable: 1.37`. KEP-4317 metadata says `stage: beta`, `latest-milestone: v1.36`, `alpha: v1.34`, `beta: v1.35`, and an empty stable milestone.

### AKS implication

Even if `ClusterTrustBundle` lands stable upstream in v1.37, the full Substrate-native path still depends on `PodCertificateRequest`. Because `PodCertificateRequest` has no recorded stable milestone yet, do not plan near-term AKS production on native support unless AKS explicitly exposes it. _[inferred]_

Live API discovery should win over timeline speculation:

```
kubectl api-resources | grep -E 'clustertrustbundles|podcertificaterequests'
kubectl explain clustertrustbundle
kubectl explain podcertificaterequest
kubectl get --raw /apis/certificates.k8s.io/v1beta1 | jq '.resources[].name'
```

_[Bayesian rule 5: overturn conditions]_ Revisit the recommendation if AKS release notes explicitly announce both APIs, or if a target AKS cluster exposes both resources in API discovery.
