# Service Account Key Rotation
Many vendors recommend regular rotation of the key pair used to sign your service account tokens.  VKS can leverage the standard Azure process for self-managed clusters with a few slight adjustments.

## Reference Documentation
[Azure AD Workload Identity - Service Account Key Rotation for Self-Managed Clusters Documentation](https://azure.github.io/azure-workload-identity/docs/topics/self-managed-clusters/service-account-key-rotation.html#key-rotation)
- Follow the proccess outlined in the Azure AD Workload Identity - Service Account Key Rotation for Self-Managed Clusters documentation linked above except for the changes listed under the **VKS Specific Changes** section below.

## VKS Specific Changes

Make the following adjustments to the steps lised in the official Azure AD Workload Idenity documentation linked above.

**Step 2 - Back Up Old Key Pair and Distrute New Key Pair section:** 
- Adjust the jump daemonset `.spec.template.spec.nodeSelector` to add a node-selector to only run on control-plane nodes.  
- Adjust the jump daemonset `.spec.template.spec.tolerations` to add a toleration to match VKS control-plane nodes.
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
- An example jump-daemonset-vks.yaml manifest is provided in this repository for reference

**Step 4 - Key Rotation**
- When you update the  `/etc/kubernetes/manifests/kube-apiserver.yaml` manifest it will trigger a restart of the kube-apiserver pod.  This may disconnect you from the cluster node(s) temporarily.  Just reconnect when the kube-apiserver is back running and make sure the files are updated on all nodes.
- I noted in testing that the `vsphere-csi` pods in the `vmware-system-csi` namespace went into `CrashLoopBackOff` after the API server restarted.  The pods all recovered after a few minutes.  In any pods are stuck in CLBO you can manually delete them 1 at a time and they should come back to running state.

## Post Update VKS - Update cluster secret
The intial SA secret was created as part of the cluster creation manifest and uses a specific naming convention `{clustername}-sa`.  In our example this secret was called `test-svc-cluster-330-sa`.  VKS cluster operators look for a secret with name to allow for overriding the SA keypair.  VKS clusters use rolling upgrades to handle cluster LCM operations.  In the case where a VKS cluster CP nodes need to be replaced, this `{clustername}-sa` secret will again be referenced to create the `sa.key` and `sa.pub` files on the control plane nodes.  We need to make sure it is updated after the key rotation process is complete.  Note: updating this secret will not cause the control-plane nodes execute a rolling update.

Once the Azure AD Workload Identity - Service Account Key Rotation for Self-Managed Clusters Documentation process is completed and verified; update the {clustername}-sa secret in the vsphere namespace.  This shold be completed as soon as possible after updating the keys.

1. Change kube context to the Supervisor cluster
2. Verify the name of the current secret (we created our vks cluster in the test-ns namespace)
```
kubectl get secret -n test-ns |grep sa

# Output
test-svc-cluster-330-sa
```
3. Set an ENV variable for secret name and namespace
```
export SECRET_NAME="test-svc-cluster-330-sa"
export NAMESPACE="test-ns"
```
4. Backup old secret
```
kubectl get secret $SECRET_NAME -n $NAMESPACE -oyaml > $SECRET_NAME-backup.yaml
```
5. Create Patch File with sa-new.pub and sa-new.key values
```
cat <<EOF > patch-sa.yaml
stringData:
  tls.crt: |
$(sed 's/^/    /' sa-new.pub)
  tls.key: |
$(sed 's/^/    /' sa-new.key)
EOF
```
6. Verify patch-sa.yaml is correctly formated
```
cat patch-sa.yaml
```
7. Patch the existing {clustername}-sa secret
```
kubectl patch secret $SECRET_NAME \
  -n $NAMESPACE \
  --patch-file patch-sa.yaml
```
8. Quick check to verify the secret has the new values - **Output from both commands should match**
```
# Check local sa-new.pub
openssl rsa -in sa-new.key -outform DER 2>/dev/null | md5sum
```
```
# Check Secret 
kubectl get secret $SECRET_NAME -n $NAMESPACE -o jsonpath='{.data.tls\.key}' | \
base64 -d | openssl rsa -outform DER 2>/dev/null | md5sum
```