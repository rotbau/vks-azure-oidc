# Service Account Key Rotation
Many vendors recommend regular rotation of the key pair used to sign your Kubernetes service account tokens. VKS leverages the standard Azure process for self-managed clusters with a few slight adjustments to accommodate the VKS architecture.

## Reference Documentation
Before proceeding, familiarize yourself with the official guide:
[Azure AD Workload Identity - Service Account Key Rotation for Self-Managed Clusters Documentation](https://azure.github.io/azure-workload-identity/docs/topics/self-managed-clusters/service-account-key-rotation.html#key-rotation)

Follow the process outlined in the official Azure documentation, substituting the specific VKS variations listed below.

## VKS Specific Changes

Make the following adjustments to the official steps during execution.

### Step 2 - Back Up Old Key Pair and Distrute New Key Pair
 When deploying the jump DaemonSet, you must ensure it targets the VKS control plane nodes correctly.

1. Adjust the jump daemonset `.spec.template.spec.nodeSelector` to add a node-selector to only target control-plane nodes.  
2. Adjust the jump daemonset `.spec.template.spec.tolerations` to match VKS control-plane taints.
```
    spec:
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master # Added for compatibility with older VKS versions
        operator: Exists
        effect: NoSchedule
```
ℹ️ Note: An example reference manifest named jump-daemonset-vks.yaml is provided in this repository.

### Step 4 - Key Rotation

⚠️ Warning: Updating the /etc/kubernetes/manifests/kube-apiserver.yaml manifest triggers an immediate restart of the kube-apiserver pod. This will temporarily disconnect you from the cluster node(s). Once the API server recovers, reconnect and verify the files are updated across all remaining nodes.

⚠️ CSI Impact Alert: During testing, vsphere-csi pods in the vmware-system-csi namespace entered a CrashLoopBackOff state following the API server restart. They typically self-recover within a few minutes. If any CSI pods remain stuck, manually delete them one at a time to force a clean restart.

## Post Update VKS - Update cluster secret
The initial Service Account (SA) secret is generated during cluster creation and follows the {clustername}-sa naming convention. VKS cluster operators watch this secret to allow SA key pair overrides.

Because VKS relies on rolling upgrades for cluster Lifecycle Management (LCM), any future Control Plane node replacements will reference this exact {clustername}-sa secret to provision `sa.key` and `sa.pub`. It is critical to update this secret immediately after a key rotation.

ℹ️ Note: Updating this secret updates the secret in the vsphere namespace and will be used when new control-plane nodes are created; it will not trigger a rolling update of your active control-plane nodes.

**Step-by-Step Secret Update**

1. Switch Context: Change your kubeconfig context to the Supervisor cluster.
2. Verify the Secret Name: Confirm the exact name of your target secret (e.g., if your cluster is in the test-ns namespace):
```
kubectl get secret -n test-ns |grep sa

# Output
test-svc-cluster-330-sa
```
3. Set Environment Variables:
```
export SECRET_NAME="test-svc-cluster-330-sa"
export NAMESPACE="test-ns"
```
4. Backup the Existing Secret: Always back up before patching production components.
```
kubectl get secret $SECRET_NAME -n $NAMESPACE -oyaml > $SECRET_NAME-backup.yaml
```
5. Create the Patch File: Generate a patch containing your newly generated sa-new.pub and sa-new.key values.
```
cat <<EOF > patch-sa.yaml
stringData:
  tls.crt: |
$(sed 's/^/    /' sa-new.pub)
  tls.key: |
$(sed 's/^/    /' sa-new.key)
EOF
```
6. Verify Patch Formatting: Inspect the generated patch file to ensure proper indentation.
```
cat patch-sa.yaml
```
7. Apply the Patch:
```
kubectl patch secret $SECRET_NAME \
  -n $NAMESPACE \
  --patch-file patch-sa.yaml
```
8. Verify the Secret Update: Ensure the cryptographic hash of the secret data matches your local new key.
```
# Check local key hash
openssl rsa -in sa-new.key -outform DER 2>/dev/null | md5sum
```
```
# Check Secret cluster-side hash (the outputs shoudl match)
kubectl get secret $SECRET_NAME -n $NAMESPACE -o jsonpath='{.data.tls\.key}' | \
base64 -d | openssl rsa -outform DER 2>/dev/null | md5sum
```