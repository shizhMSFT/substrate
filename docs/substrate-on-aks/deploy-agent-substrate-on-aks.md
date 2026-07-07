# Deploy Agent Substrate on AKS

> Date: 2026-07-07 (milestone reached 2026-06-10)

## Outcome

The Agent Substrate AKS deployment path reached a successful smoke test, based on the user's live AKS run. This validates the first development-grade AKS path for provisioning, deployment, image pulls, control-plane state, and snapshot storage. The smoke test result is user-reported rather than independently rerun in this planning repo.

## What was built

- **Azure provisioning:** added `tools/setup-azure` as the Azure counterpart to `tools/setup-gcp`, covering resource provider registration, AKS cluster creation, snapshot storage, atelet Workload Identity, atelet Blob/ACR permissions, and kubelet ACR pull permissions.
- **Azure Blob snapshots:** added an atelet Azure Blob object-storage backend selected by `ATE_STORAGE_BACKEND=azure`.
- **Snapshot URI scheme:** added support for `azblob://<container>/<prefix>` alongside existing object-storage URI parsing.
- **AKS overlay:** added `manifests/ate-install/aks` and selected it with `ATE_INSTALL_AKS=true`, mirroring the existing incremental `ATE_INSTALL_KIND=true` pattern rather than introducing a broader platform abstraction.
- **Azure Workload Identity:** wired the atelet Kubernetes service account with Azure Workload Identity annotations/labels and configured the setup tool to print `AZURE_ATELET_CLIENT_ID`.
- **ACR image pulls inside atelet:** added `--azure-auth-for-image-pulls=true` so atelet's internal image pull cache can authenticate to Azure Container Registry with managed identity credentials.
- **Demo snapshot location:** the demo templates use an `ATE_STORAGE_ROOT` placeholder (substituted by the `hack/install-demo-*.sh` scripts), so Azure runs can set `ATE_STORAGE_ROOT=azblob://...` without pretending Azure containers are GCS buckets. (`ATE_STORAGE_ROOT` superseded the earlier `SNAPSHOT_LOCATION` variable.)

## Live issues found and fixes

- **Azure role assignment API version:** assigning `Storage Blob Data Contributor` failed because the old Azure Authorization SDK used an API version that does not support roles with `DataActions`. Fixed by moving setup role-assignment clients to `armauthorization/v2`.
- **Missing `ClusterTrustBundle` on AKS:** AKS did not expose the `ClusterTrustBundle` API needed by the base manifests. First workaround mounted a generated `workerpool-ca-certs` Secret instead of using `ClusterTrustBundle`.
- **Missing `podcertificaterequests` on AKS:** AKS also lacked the backing `podcertificaterequests` resource, so the `podCertificate` projected-volume path could not produce credential bundles. Final AKS dev workaround removed all `podCertificate`/`ClusterTrustBundle` usage from the AKS rendered manifests and mounted static Secrets instead.
- **Static service TLS bundle:** `hack/install-ate.sh` now creates `ate-system/servicedns-credential-bundle` from the generated service-DNS CA pool and the AKS overlay mounts it where the base manifests expect `/run/servicedns.podcert.ate.dev/credential-bundle.pem`.
- **Valkey init Job immutability:** the existing `valkey-cluster-init` Job could not be updated after its pod template changed. AKS deploy now deletes/recreates that Job before applying the rendered manifests.

## Important caveat

The static TLS Secret path is a development bootstrap, not a production identity or rotation design. It exists because the tested AKS cluster did not expose the upstream pod-certificate APIs assumed by the base manifests. Production-quality AKS support would need a proper certificate/identity strategy rather than static dev Secrets.

## Relevant substrate change areas

- AKS runtime wiring and overlay selection.
- Newer Azure authorization API for role assignments with `DataActions`.
- AKS dev workaround for missing `ClusterTrustBundle` and `podCertificate` support.
- Static TLS Secret generation and cleanup for the dev AKS path.
- Valkey init Job recreation during AKS deploy.

## Validation status

- Local validation was run during implementation: Go tests for touched setup/runtime packages, shell syntax checks, and AKS kustomize rendering checks.
- Live validation: user completed the AKS smoke test successfully.
- Remaining unknowns: this is not production-grade AKS identity; it validates the development runtime path, not long-term certificate lifecycle or rotation.
