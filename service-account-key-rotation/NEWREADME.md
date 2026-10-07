# Service Account Key Rotation for VKS Clusters

Many vendors recommend regular rotation of the key pair used to sign Kubernetes service account tokens. VKS leverages the standard Azure Workload Identity process for self-managed clusters with slight adjustments to accommodate the VKS architecture.

---

## Prerequisites

Before executing this workflow, ensure you have:
* `kubectl` installed and configured.
* Access to both the **Workload Cluster** and **Supervisor Cluster** `kubeconfig` contexts.
* Review the [Official Azure SA Key Rotation Documentation](https://azure.github.io/azure-workload-identity/docs/topics/self-managed-clusters/service-account-key-rotation.html#key-rotation).

---

## Repository Files

| File | Description |
| :--- | :--- |
| `jump-daemonset-vks.yaml` | Modified DaemonSet manifest with VKS control plane node selectors and tolerations. |
| `service-account-secret-rotation.yaml` | Sample Supervisor secret manifest for replacing `tls.crt` and `tls.key`. |
| `patch-node-label-override-not-present.yaml` | JSON Patch template to force a control plane topology re-role if override section doesn't exist. |
| `patch-node-label-override-present.yaml` | JSON Patch template to force a control plane topology re-role if override section does exist. |

---


## Option 1 - Step-by-Step Rotation Process - Offcial Azure Process

### Step 1: Distribute New Key Pair (Workload Cluster Context)

When deploying the jump DaemonSet listed in the official Azure guide, you must target the VKS control plane nodes specifically by adjusting the jump-daemonset manifest.

1. **Switch Context:** Ensure `kubectl` is pointed to your **Workload Cluster**.
2. **Configure Node Selection & Tolerations:** Ensure the DaemonSet includes the VKS control plane node selectors and tolerations:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master # Compatibility for older VKS versions
          operator: Exists
          effect: NoSchedule

```

> [!NOTE]
> You can apply the pre-configured [jump-daemonset-vks.yaml](https://www.google.com/search?q=jump-daemonset-vks.yaml) directly from this repository.

3. **Deploy the DaemonSet:** Apply the modified DaemonSet to distribute the key pairs across all control plane nodes.
4. **Follow Official Azure Steps:** Complete the key generation and distribution steps outlined in the [Official Azure Documentation](https://azure.github.io/azure-workload-identity/docs/topics/self-managed-clusters/service-account-key-rotation.html#key-rotation).

---

### Step 2: Update the Cluster Secret (Supervisor Context)

The initial Service Account secret is created during cluster provisioning using the `{clustername}-sa` naming convention. VKS relies on this secret during lifecycle management (LCM) operations.

> [!IMPORTANT]
> Updating this secret updates future node configurations in the vSphere namespace, but **it will not trigger a rolling update** of active control plane nodes on its own.

1. **Switch Context:** Change your `kubeconfig` to the **Supervisor Cluster**.
2. **Verify and Set Environment Variables:**

```bash
export NAMESPACE="test-ns"
export SECRET_NAME=$(kubectl get secret -n $NAMESPACE -o name | grep sa | cut -d/ -f2)

# Verify output
echo "Target Secret: $SECRET_NAME in $NAMESPACE"

```

3. **Back Up Existing Secret:**

```bash
kubectl get secret $SECRET_NAME -n $NAMESPACE -o yaml > ${SECRET_NAME}-backup.yaml

```

4. **Patch the Secret:**

**Option A — Automated Patch File:**

```bash
cat <<EOF> patch-sa.yaml
stringData:
  tls.crt: |
$(sed 's/^/    /' sa-new.pub)
  tls.key: |
$(sed 's/^/    /' sa-new.key)
EOF

kubectl patch secret $SECRET_NAME -n $NAMESPACE --patch-file patch-sa.yaml

```

**Option B — Direct Manifest Apply:**
Edit [service-account-secret-rotation.yaml](service-account-secret-rotation.yaml) with your new `tls.crt` and `tls.key` values, then apply:

```bash
kubectl apply -f service-account-secret-rotation.yaml -n $NAMESPACE

```

---

## Option 2 - Key Rotation Process using VKS Specific Option (Alternate to option 1)

### Step 1 - Update service-account-secret-rotation with your new sa.pub and sa.key values

1. Edit [service-account-secret-rotation.yaml](service-account-secret-rotation.yaml) with your new `tls.crt` and `tls.key` values
2. Change kubectl context to your Supervisor Cluster
3. Apply service-account-secret-rotation.yaml with updated tls.crt and tls.key values
```
kubectl apply -f service-account-secret-rotation.yaml
```

### Step 2: Trigger Control Plane Rolling Update (Supervisor Context)

You need to force VKS to re-role the control plane nodes so they adopt the updated secret. You can patch the control plane nodes to apply a timestamp variable override under `spec.topology.controlPlane`.

**Option A — JSON Merge Patch (Recommended):**
Works regardless of whether `spec.topology.controlPlane.variables` already exists.

```bash
export CLUSTER_NAME="cluster-v33"

kubectl patch cluster $CLUSTER_NAME -n $NAMESPACE --type=merge -p "
spec:
  topology:
    controlPlane:
      variables:
        overrides:
        - name: node
          value:
            labels:
              sa-token-rotation: \"$(date +%s)\"
"

```

**Option B — JSON Patch (If appending to an existing `overrides` list):**

```bash
sed "s/TIMESTAMP/$(date +%s)/" patch-node-label-override-present.yaml | kubectl patch cluster $CLUSTER_NAME -n $NAMESPACE --type=json --patch-file /dev/stdin

```

> [!NOTE]
> Monitor the rolling upgrade in the Supervisor cluster until all control plane nodes report `Ready`.