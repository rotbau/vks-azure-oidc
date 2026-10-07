# Service Account Key Rotation
Many vendors recommend regular rotation of the key pair used to sign your Kubernetes service account tokens. VKS leverages the standard Azure process for self-managed clusters with a few slight adjustments to accommodate the VKS architecture.

## OFFICIAL AZURE Process to rotate Service Account Keys
Before proceeding, familiarize yourself with the official guide:
[Azure AD Workload Identity - Service Account Key Rotation for Self-Managed Clusters Documentation](https://azure.github.io/azure-workload-identity/docs/topics/self-managed-clusters/service-account-key-rotation.html#key-rotation)

Follow the process outlined in the official Azure documentation, substituting the specific VKS variations listed below.

## VKS Specific Changes

Make the following adjustments to the official steps during execution.

### Step 1 - Back Up Old Key Pair and Distribute New Key Pair
 When deploying the jump DaemonSet listed on the official website, you must ensure it targets the VKS control plane nodes correctly.

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
ℹ️ Note: An example reference manifest named [jump-daemonset-vks.yam](jump-daemonset-vks.yaml) is provided in this repository.

3. Deploy the jump-daemonset pod incorporating the VKS Specific adjustments to adsure the pod lands on the control-plane nodes
4. Follow the remaining steps on the [Official Azure Key Rotation Documentation](https://azure.github.io/azure-workload-identity/docs/topics/self-managed-clusters/service-account-key-rotation.html#key-rotation)

### Step 2 - Post Key Rotation - Update the Cluster Secret created in the vSphere Namespace
The initial Service Account (SA) secret is generated during cluster creation and follows the {clustername}-sa naming convention. VKS cluster operators watch this secret to allow SA key pair overrides.

Because VKS relies on rolling upgrades for cluster Lifecycle Management (LCM), any future Control Plane node replacements will reference this exact {clustername}-sa secret to provision `sa.key` and `sa.pub`. It is critical to update this secret immediately after a key rotation.

ℹ️ Note: Updating this secret updates the secret in the vsphere namespace and will be used when new control-plane nodes are created; it will not trigger a rolling update of your active control-plane nodes or affect the running cluster.

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
5. Patch the Existing Secret

**Option 1 - Create and Apply Patch File**
```
cat <<EOF > patch-sa.yaml
stringData:
  tls.crt: |
$(sed 's/^/    /' sa-new.pub)
  tls.key: |
$(sed 's/^/    /' sa-new.key)
EOF
```
- Verify Patch Formatting: Inspect the generated patch file to ensure proper indentation.
```
cat patch-sa.yaml
```
- Apply the Patch:
```
kubectl patch secret $SECRET_NAME \
  -n $NAMESPACE \
  --patch-file patch-sa.yaml
```
**Option 2 - Update Existing Secret using YAML**
- Reference the example [service-account-secret-rotation.yaml](service-account-secret-rotation.yaml)
- Update tls.crt and tls.key with the new values
- Apply the service-account-secret-rotation.yaml file to the Supervisor context.
```
kubectl apply -f service-account-secret-rotation.yaml
```










8. Force Control-plane nodes to get recreated.  Existing clusters need to get redeployed to insert new Keys onto the node.

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