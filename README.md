# VKS Cluster Class and Cluster for Custom Service Account Provider

These files can be used to add a custom OIDC endpoint for service account authorization and role assumption on VKS clusters

Disclaimer: Use at your own risk. This project is provided "as is" without any warranty of any kind, either expressed or implied. The author assumes no liability for any damages or data loss caused by the use of this configuration. This is not an official product and is not supported by any organizatio

## Prerequistes
- Supervisor 1.28 or later
- VKS version 3.3.0 or later (Tanzu Kubernetes Grid Service)
- vSphere Namespace access with at least edit permissions
- Jumpbox with jq and kubectl, kubectl-vsphere plugin installed

## Files
- svc-account-issuer-custom-class-3.3.0.yaml   - This is the custom cluster class that exposes the service-account-issuer variable
- test-svc-cluster-3.3.0.yaml - VKS cluster manifest with cluster secret that leverage the custom cluster class
- patch-issuer.yaml - json patch file that will update the service-account-issuer value

## Proccedure 
**Note:** All kubectl commands are run from the Supervisor context

1. Modify .metaname.namepace in svc-account-issuer-custom-class-3.3.0.yaml with your vSphee namespace
2. Create the custom cluster class
```
kubectl apply -f svc-account-issuer-custom-class-3.3.0.yaml 
```
3. Modify test-svc-cluster-3.3.0.yaml with the correct values for **namespace**, **vmClass**, **storageClass** and **defaultStorageClass** that align with your enviornment.  You can use the following commands to determine storageClass and vmClass available to you.
```
# vmClass spec.topology.variables[name="vmClass"].value

kubectl get vmclass -n <vsphere-namespace>
NAME                 CPU   MEMORY
best-effort-large    4     16Gi
best-effort-medium   2     8Gi
best-effort-small    2     4Gi
best-effort-xlarge   4     32Gi
best-effort-xsmall   2     2Gi

# storageClass spec.topology.variables[name="storageClass"].value

kubectl describe ns <vsphere-namespace>

# Example Output: storage class is vsan-default-storage-policy
  Name:                                                                     test-ns-storagequota
  Resource                                                                  Used  Hard
  --------                                                                  ---   ---
  vsan-default-storage-policy.storageclass.storage.k8s.io/requests.storage
```
4. Create VKS Cluster
```
kubectl apply -f test-svc-cluster-3.3.0.yaml
```
5. Verify cluster correctly builds
```
kubectl get clusters -n <vsphere-namespace>
kubectl get kcp -n <vsphere-namespace>
kubectl get md -n <vsphere-namespace>
kubectl get ms -n <vsphere-namespace>
kubectl get md -n <vsphere-namespace>
```
6. Check Default Service Account Issuer Value
```
kubectl get kcp -n <vpsphere-namespace>

kubectl get kcp test-svc-cluster-330-xxxxx -n <vsphere-namespace> -o json | jq -r '
  .spec.kubeadmConfigSpec.clusterConfiguration.apiServer.extraArgs[] 
  | select(.name == "service-account-issuer") 
  | .value'

# Output should display
https://kubernetes.default.svc.cluster.local
```


## Patching
One the OIDC endpoint has been created we can patch the cluster with the new value.

A merge type patch fails because the cluster validating webhook expects all values under the .spec.topology.variables to be provided in the patch.  This is undesirable and may cause the entire cluster to re-roll nodes instead of just the control plane nodes.  To get around this we use a 2 part process.  We determine the correct index of the service-account-issuer variable and then patch it directly.

1. Determine Index Location
```
kubectl get cluster test-svc-cluster-330 -n <vsphere-namespace> -o jsonpath='{range .spec.topology.variables[*]}{.name}{"\n"}{end}' | grep -n "serviceAccountIssuer"
1:serviceAccountIssuer
```
We need to correct the index number (x-1) becaue Grep starts at 1 and Kubernetes starts at 0.  So index 1 becomes 0

2. Create a Patch File (or update example patch in this repository)
```
- op: replace
  path: /spec/topology/variables/0/value                             # Update the index for correct value
  value: "https://login.microsoftonline.com/tenant123/v2.0"          # update with OIDC endpoint (no terminal /)
```

3. Patch Cluster
```
 k patch cluster test-svc-cluster-330 \
 -n <vsphere-namespace> \
 --type json \
 --patch-file patch-issuer.yaml
 ```

 Note: This will trigger a rolling update of the control plane node(s) in the cluster

## Verification

1. Get kcp object name
```
kubectl get kcp -n <vsphere-namesapce> 
```

2. Veriy Service-Account-Issuer is Updated
```
kubectl get kcp test-svc-cluster-330-xxxxx -n test-ns -o json | jq -r '
  .spec.kubeadmConfigSpec.clusterConfiguration.apiServer.extraArgs[] 
  | select(.name == "service-account-issuer") 
  | .value'

# Output Should Show Value from Patch
```

3. Check Well-Known ID for Cluster (from VKS cluster context)
```
 kubectl get --raw /.well-known/openid-configuration |jq
```

