# Custom Cluster Class 3.3 Workaround

This workaround defines the variables in the cluster class but doesn't set a value.  The patch then adds the "new" endpoint which will sign all new tokesn but also retain the default so existing tokens are trusted.

## Recommended Approach: variable-driven optional patch
1. Create a custom ClusterClass with an inline variable for the external issuer URL (externalServiceAccountIssuerURL, optional), the default patch, and one always-on patch that appends the kubeadm default --service-account-issuer via kubeadm's native patches directory (/run/kubeadm/patches) — but only when the variable is set (enabledIf), to avoid duplicating kubeadm's own implicit default.  [Example Cluster-Class](custom-cluster-class-3.3.yaml)
2. Create the Cluster without setting externalServiceAccountIssuerURL — both optional patches stay disabled, and the cluster comes up with just kubeadm's own implicit default issuer. [Example Cluster](cluster-v33.yaml)
3. SSH to control-plane Node of new cluster when cluster build in completed
```
# Get VMs in vSphere Namespace and record IP of control-plane Node.
kubectl get virtualmachines -n <VSPHERE_NAMESPACE> -o wide

NAME                                   POWER-STATE   CLASS               IMAGE                   PRIMARY-IP4   AGE
cluster-v33-5z5tj-46fqh                PoweredOn     best-effort-medium   vmi-53f6ec4e14352a2ee   10.0.102.17   97m    <---- control-plane Node
cluster-v33-worker-wqfst-x4m9x-7vfcq   PoweredOn     best-effort-medium   vmi-53f6ec4e14352a2ee   10.0.102.15   149m

# Decrypt the SSH Password for the Cluster
kubectl get secret <CLUSTER_NAME>-ssh-password -n <VSPHERE_NAMESPACE> -o jsonpath='{.data.ssh-passwordkey}' | base64 -d

Output example: (password includes the = sign but excludes the %)
5vGlQ5hV16iHDkHJyve9K2Kuil7fgvLLauh7tyPmeZPI=%  

# SSH to control-plane Node
ssh vmware-system-user@<WORKER_NODE_IP>

# Change to Root
sudo -i
```
4. Verified on the resulting control-plane node — only the kubeadm default issuer is defined, no duplicate:
```
grep service-account /etc/kubernetes/manifests/kube-apiserver.yaml

# Output
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
```
5. Once your exteral issuer URL is available, patch the cluster.  Replace "value" with you actual endpoint.
```
kubectl patch cluster cluster-v34 -n test-ns --type=json -p '[{"op":"add","path":"/spec/topology/variables/-","value":{"name":"externalServiceAccountIssuerURL","value":"https://login.microsoftonline.com/tenant123/v2.0"}}]'
```
6. Control Plane Node(s) will re-roll.  Wait for the cluster update to be completed
7. Once the new Control-plane nodes are available, verify settings.  Repeat step 3 to SSH to new control-plane node.  Note: IP address will be different.
```
grep service-account /etc/kubernetes/manifests/kube-apiserver.yaml

# Output
    - --service-account-issuer=https://login.microsoftonline.com/tenant123/v2.0
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
```
New issuer is added first (signs new tokens), kubeadm default stays second (accepted-only) — existing tokens remain valid. Only the Cluster's variable was touched; the ClusterClass itself was never modified.