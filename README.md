# External Service Account Issuer for VKS Clusters

This project defines variables in a custom `ClusterClass` to allow setting an external service account issuer URL without invalidating pre-existing tokens. 

By leveraging a variable-driven optional patch, new tokens are signed by the newly added external endpoint while retaining the default `kubeadm` issuer as an accepted fallback.

## Process Overview

### How It Works
1. **Inline Variable Definition**: An optional `externalServiceAccountIssuerURL` variable is declared in the `ClusterClass`.
2. **Conditional Patch Execution**: An `enabledIf` condition appends the `kubeadm` default `--service-account-issuer` via `kubeadm`'s native patch directory (`/run/kubeadm/patches`) only when the variable is provided, preventing duplicate default flags.
3. **Non-Disruptive Cluster Update**: Existing tokens remain trusted, and modifications are confined to the `Cluster` spec level—the `ClusterClass` itself requires no ongoing edits.

---

## Prerequisites

- **Supervisor**: Version 1.28 or later
- **VKS**: Version 3.3.0 or later (supports VKS 3.3 – 3.8)
- **Permissions**: vSphere Namespace access with at least `edit` permissions
- **Jumpbox**: Configured with `jq`, `kubectl`, and the `kubectl-vsphere` plugin

---

## Repository Structure & Manifests

```text
.
├── clusterclass/
│   ├── custom-cluster-class-3.3.yaml
│   ├── custom-cluster-class-3.4.yaml
│   └── custom-cluster-class-3.7.yaml
├── clusters/
│   ├── cluster-v33.yaml
│   ├── cluster-v34.yaml
│   └── cluster-v37.yaml
├── patches/
│   ├── patch-issuer.yaml
├── secret-rotation/
│   ├── README.md
│   ├── patch-node-label.yaml
│   └── service-account-secret-rotation.yaml
└── test-workloads/
    └── dummy-pod.yaml

```

---

## Deployment & Configuration Workflow

### Step 1: Create Custom Cluster Class

Custom Cluster Classes are created in the target vSphere Namespace where you plan to deploy your VKS cluster (e.g., `test-ns`).

1. Set your `kubectl` context to the Supervisor Cluster.
2. Select and edit the appropriate manifest in `clusterclass/` matching your Supervisor's VKS Service version (**Do not modify any additional settings**).
3. Apply the `ClusterClass` manifest:
```bash
kubectl apply -f clusterclass/custom-cluster-class-x.y.z.yaml

```

### Step 2: Deploy the VKS Cluster

Deploy a VKS cluster referencing the custom `ClusterClass` created in Step 1.

1. Ensure your `kubectl` context is set to the Supervisor Cluster.
2. Edit the appropriate `clusters/cluster-vXY.yaml` file:
* Change the cluster name (optional).
* Ensure the `namespace` matches your target vSphere namespace.
* Adjust control plane and node pool replica counts as needed.
* *Do not modify `clusterclassRef` or other system components unless necessary.*

3. Deploy the cluster **without** setting `externalServiceAccountIssuerURL`. The cluster will initialize using `kubeadm`'s default issuer [Example Cluster](clusters/cluster-v33.yaml)
```bash
kubectl apply -f clusters/cluster-vXY.yaml

```

### Step 3: Validate Initial Cluster Settings (Optional)

1. Obtain the IP address of the Control-Plane Node from the Supervisor context:
```bash
kubectl get virtualmachines -n <VSPHERE_NAMESPACE> -o wide

```

2. Decrypt the SSH password for the cluster:
```bash
kubectl get secret <CLUSTER_NAME>-ssh-password -n <VSPHERE_NAMESPACE> -o jsonpath='{.data.ssh-passwordkey}' | base64 -d

```

3. SSH into the Control-Plane Node and switch to root:
```bash
ssh vmware-system-user@<CONTROL_PLANE_NODE_IP>
sudo -i

```

4. Verify that only the default `kubeadm` issuer is active:
```bash
grep service-account /etc/kubernetes/manifests/kube-apiserver.yaml

```

*Expected output:*
```yaml
- --service-account-issuer=[https://kubernetes.default.svc.cluster.local](https://kubernetes.default.svc.cluster.local)
- --service-account-key-file=/etc/kubernetes/pki/sa.pub
- --service-account-signing-key-file=/etc/kubernetes/pki/sa.key

```

5. Authenticate to the workload cluster via `kubectl vsphere login` or `vcf context create`, switch contexts, and verify node health:
```bash
kubectl config use-context cluster-vXY
kubectl get nodes

```

### Step 4: Install Azure Arc Software

Follow [Microsoft Azure Official Documentation](https://azure.github.io/azure-workload-identity/docs/installation/self-managed-clusters.html) to install the required Azure AD Workload Identity components on your workload cluster before updating the issuer URL.

### Step 5: Patch Cluster with External Issuer

Once your external issuer URL is available, update the cluster definition.

1. **Option A:** Apply an inline JSON patch:
```bash
kubectl patch cluster cluster-v33 -n <VSPHERE_NAMESPACE> --type=json \
  -p '[{"op":"add","path":"/spec/topology/variables/-","value":{"name":"externalServiceAccountIssuerURL","value":"[https://login.microsoftonline.com/tenant123/v2.0](https://login.microsoftonline.com/tenant123/v2.0)"}}]'

```

**Option B:** Use a patch file [Example patch-issuer.yaml](patches/patch-issuer.yaml)
```yaml
- op: add
  path: /spec/topology/variables/-
  value:
    name: externalServiceAccountIssuerURL
    value: [https://login.microsoftonline.com/tenant123/v2.0](https://login.microsoftonline.com/tenant123/v2.0)

```

Apply the patch file:
```bash
kubectl patch cluster <ClusterName> -n <VSPHERE_NAMESPACE> --type=json --patch-file patch.yaml

```

2. Wait for the Control Plane node(s) to redeploy with the updated configuration.
3. SSH into the new Control-Plane Node (note that the IP address may have changed) and confirm the settings:
```bash
grep service-account /etc/kubernetes/manifests/kube-apiserver.yaml

```

*Expected output:*
```yaml
- --service-account-issuer=[https://login.microsoftonline.com/tenant123/v2.0](https://login.microsoftonline.com/tenant123/v2.0)
- --service-account-key-file=/etc/kubernetes/pki/sa.pub
- --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
- --service-account-issuer=[https://kubernetes.default.svc.cluster.local](https://kubernetes.default.svc.cluster.local)

```

> **Note:** The new external issuer is listed first (used to sign new tokens), while the default `kubeadm` issuer remains second (accepted for existing tokens). The `ClusterClass` itself remains unmodified throughout this process.

---

## Service Account Key Rotation

Reference the [Service Account Key Rotation](service-account-key-rotation/README.md) section for details on how to rotate the secret containing the TLS key pairs that sign token requests.

---

## Disclaimer

Use at your own risk. This project is provided "as is" without warranty of any kind, express or implied. The author assumes no liability for damages or data loss resulting from the use of this configuration. This is not an official product and is not supported by any organization.