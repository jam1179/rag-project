# AKS Deployment Design — MCP RAG Server and Chat Demo

**Roadmap:** Iteration 7.9.7.7 — AKS / EKS Deployment Design  
**Status:** AKS manifest and credential-configuration checks passed locally; **Azure infrastructure has not been provisioned or runtime-tested**.  
**Scope:** Infrastructure contracts and deployment validation only. Do not apply this overlay to the existing Kind cluster.

## 1. Architecture and deployment boundaries

The AKS overlay (`k8s/overlays/aks`) extends the portable manifests in `k8s/base`. The MCP RAG Server remains a private `ClusterIP` service. The Streamlit Chat Demo is intended to be exposed through an HTTPS ingress, which is **not yet implemented by this overlay**.

| Workload | Kubernetes ServiceAccount | Azure identity | Key Vault CSI class |
| --- | --- | --- | --- |
| MCP RAG Server | `mcp-rag-server-sa` | Dedicated user-assigned managed identity | `mcp-rag-server-kv` |
| Streamlit Chat Demo | `mcp-rag-chat-demo-sa` | Separate user-assigned managed identity | `mcp-rag-chat-demo-kv` |

Both workloads mount a Key Vault secret as a read-only file at `/mnt/secrets/gemini-api-key` and set `GEMINI_API_KEY_FILE` to that path. The Python settings layer reads the file at application initialization. The AKS overlay must not require the Kind-only Kubernetes Secrets or synchronize the Key Vault secret into a Kubernetes Secret.

The MCP Server is deliberately a **single replica** using a `ReadWriteOnce` LanceDB PVC and the `Recreate` Deployment strategy. This avoids concurrent writes but causes downtime during replacement or controlled restart.

## 2. AKS cluster prerequisites

The Azure infrastructure team must provide:

1. An AKS cluster with the **OIDC issuer** and **Microsoft Entra Workload Identity** enabled.
2. The **Secrets Store CSI Driver** and **Azure Key Vault provider** installed and operational (for example, using the supported AKS Key Vault secrets provider add-on). Confirm compatibility with the selected AKS/Kubernetes version.
3. A functional CSI driver node installation on nodes where the application Pods can be scheduled.
4. Network access from the cluster to Microsoft Entra token endpoints, Azure Key Vault, and Azure Container Registry, with DNS, firewall, and private endpoint configuration as applicable.
5. An ingress controller and TLS certificate solution for public HTTPS access to Streamlit. **No ingress manifest is included yet.**

Manifest rendering cannot establish that these prerequisites exist.

## 3. Workload identities and federation

Provision **two distinct user-assigned managed identities**, one per workload. Replace the illustrative client IDs in both `service-accounts.yaml` and `secret-provider-classes.yaml` with their real values.

| Federated credential property | MCP Server | Chat Demo |
| --- | --- | --- |
| Issuer | AKS cluster OIDC issuer URL | Same AKS OIDC issuer URL |
| Subject | `system:serviceaccount:mcp-rag:mcp-rag-server-sa` | `system:serviceaccount:mcp-rag:mcp-rag-chat-demo-sa` |
| Audience | `api://AzureADTokenExchange` | `api://AzureADTokenExchange` |
| Client ID | MCP Server identity client ID | Chat Demo identity client ID |

The `azure.workload.identity/client-id` annotation on each ServiceAccount and the `clientID` parameter of its corresponding `SecretProviderClass` must refer to the **same identity**. Each Pod template must carry `azure.workload.identity/use: "true"` and reference the correct ServiceAccount.

The UUIDs currently in the manifests (`00000000-0000-0000-0000-000000000001` and `00000000-0000-0000-0000-000000000002`) are **nonfunctional placeholders**. The tenant ID (`00000000-0000-0000-0000-000000000003`) is also a placeholder.

> Federation, token exchange, webhook injection, and identity authorization cannot be verified by Kustomize. These require a configured AKS environment.

## 4. Azure Key Vault and CSI secret delivery

Create or select an Azure Key Vault and store the Gemini credential as a Key Vault **secret** named `gemini-api-key`. Replace the placeholder Key Vault name (`my-rag-keyvault`) and tenant ID in both `SecretProviderClass` resources.

Grant each managed identity the minimum permission needed to **read the intended secret**. For a Key Vault configured with Azure RBAC, use an appropriately scoped role such as **Key Vault Secrets User**. If the vault uses access policies instead, configure equivalent secret `get` permission. Avoid broad vault-administrator privileges.

The overlay uses:

- `provider: azure` and `usePodIdentity: "false"` with the workload identity client ID.
- `objectName: gemini-api-key`, `objectType: secret`, and `objectAlias: gemini-api-key`.
- `filePermission: "0444"` to permit non-root UID 999 to read the mounted file.
- A read-only CSI volume mounted at `/mnt/secrets` in each application container.
- **No `secretObjects` section**: do not synchronize secrets into native Kubernetes Secrets.
- ConfigMap-only `envFrom` configuration in the AKS Deployments; the base's Kind-specific `secretRef` is removed by the overlay.

**Permission trade-off:** `0444` makes the file readable by any process with access to that mounted path inside the container. It does not, by itself, mount the secret into unrelated Pods. Consider narrower file ownership/permission controls during production hardening, and verify actual read access as UID 999 on AKS.

**Rotation contract:** The CSI provider may update mounted secret content when rotation is enabled, but the current applications read the Gemini API key during startup. After a key change, perform a **controlled restart** of the affected Deployment(s). Automatic hot reload is deferred. With a single MCP Server replica and `Recreate`, a restart incurs temporary downtime.

## 5. Container images and registry

The overlay currently rewrites images to:

- `myregistry.azurecr.io/mcp-rag-server:0.1.0`
- `myregistry.azurecr.io/mcp-rag-chat-demo:0.1.0`

`myregistry.azurecr.io` is a **placeholder**. Publish versioned images to the actual Azure Container Registry and grant the AKS kubelet identity permission to pull them using the registry's supported authorization model. Application workload identities for Key Vault access are separate from the kubelet image-pull identity.

**Release blocker:** The Docker image security review previously identified **59 HIGH and 1 CRITICAL Debian OS image vulnerabilities** (and no HIGH/CRITICAL Python dependency vulnerabilities at that checkpoint). These remain unresolved; successful AKS manifest validation **does not authorize a production release**.

## 6. Persistent storage

The MCP Server uses PVC `mcp-rag-lancedb` at `/data/lancedb`, with:

- Requested capacity: **5Gi**.
- Access mode: **ReadWriteOnce**.
- StorageClass name: **`managed-csi-premium-retain`**.
- Deployment replicas: **1**.
- Deployment update strategy: **Recreate**.

`managed-csi-premium-retain` is an **infrastructure contract, not a verified built-in StorageClass**. Provision and validate an Azure Disk CSI StorageClass with suitable disk SKU, topology, expansion policy, and the intended `Retain` reclaim policy. Ensure the PVC can bind and reattach after Pod rescheduling. A retained disk does not replace backups, restore procedures, or application-level recovery validation.

The existing Kind PVC and base manifests must remain unchanged.

## 7. Networking and exposure

- **MCP RAG Server:** private Kubernetes `ClusterIP`; not directly internet-facing.
- **Chat Demo:** intended public HTTPS endpoint through an ingress/application gateway and TLS termination; ingress manifests and access policies remain future work.
- **Egress:** Gemini API access and any required Azure identity/Key Vault/registry endpoints must be allowed.
- Define ingress authentication, WAF/rate limits, network policies, and external exposure controls before production deployment.

Do not assume that the presence of a `ClusterIP` alone enforces application-level authorization or network isolation.

## 8. Local manifest validation (no Azure provisioning)

From repository root `C:\study\ai-projects\rag-project` in PowerShell:

```powershell
# Render AKS overlay using UTF-8 (avoid Windows PowerShell 5.1 UTF-16 redirection)
kubectl kustomize k8s/overlays/aks | Out-File -FilePath aks-rendered.yaml -Encoding utf8

# Validate rendered Deployments, identities, CSI classes and secret references
uv run --no-project --with pyyaml python scripts/validate_aks_manifests.py

# Confirm portable base manifests are unchanged
git diff -- k8s/base
```

If your PowerShell produces a UTF-8 BOM, the validation script should open the file with `encoding="utf-8-sig"`. The `--no-project --with pyyaml` command does not convert the repository root into a uv project or add a dependency to either application.

**Latest user-confirmed result:**

```text
PASS: mcp-rag-server Key Vault identity and secret configuration
PASS: mcp-rag-chat-demo Key Vault identity and secret configuration
PASS: mcp-rag-server
PASS: mcp-rag-chat-demo
AKS CSI manifest validation PASSED
```

These checks establish **rendered manifest consistency only**. They do not establish that the cluster, identities, Key Vault, CSI driver, storage, or ingress exist or work.

## 9. Real-AKS deployment acceptance checklist (deferred)

Before declaring an AKS deployment operational, verify:

- [ ] AKS OIDC issuer, workload identity webhook, and CSI provider are installed and healthy.
- [ ] Real identity client IDs, tenant ID, Key Vault name, and ACR hostname replace placeholders.
- [ ] Both federated credentials match the exact namespace, ServiceAccount subject, issuer, and audience.
- [ ] Both identities can read only their authorized Key Vault secrets.
- [ ] CSI mounts succeed and `gemini-api-key` is readable by UID 999; do not print its value.
- [ ] No native Kubernetes Secret is created or required for Gemini credentials.
- [ ] Images pull successfully and image vulnerabilities are remediated or formally dispositioned.
- [ ] The Azure Disk PVC binds, data persists across restart, and storage recovery is exercised.
- [ ] Streamlit HTTPS ingress, certificate, authentication, and network exposure controls are validated.
- [ ] Both applications start, pass probes, and complete a grounded RAG end-to-end request.
- [ ] Key rotation followed by controlled restart is tested, including expected downtime.
- [ ] Logs and diagnostics do not expose credential contents.

## 10. Scope and follow-up roadmap

**Completed locally:** AKS overlay Kustomize rendering, workload identity annotation/label consistency, CSI volume and environment wiring, SecretProviderClass consistency, no Kubernetes Secret synchronization, and configured `filePermission: "0444"`.

**Not completed:** Azure resource provisioning, actual federated authentication, runtime CSI mount and file-permission checks, image security remediation, HTTPS ingress, or cloud end-to-end testing.

**Next roadmap items:** Finish the EKS design within **7.9.7.7**; cover failure recovery and persistence in **7.9.7.8**, Kubernetes regression and closure in **7.9.7.9**, and gateway, authorization, observability, scaling, CI/CD, and production readiness in subsequent canonical iterations. Automatic credential hot reload remains deferred.
