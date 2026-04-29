# VKS Cluster Class and Cluster for Custom Service Account Provider

These files can be used to add a custom OIDC endpoint for service account authorization and role assumption on VKS clusters

Disclaimer: Use at your own risk. This project is provided "as is" without any warranty of any kind, either expressed or implied. The author assumes no liability for any damages or data loss caused by the use of this configuration. This is not an official product and is not supported by any organization


## Prerequistes
- Supervisor 1.28 or later
- VKS version 3.3.0 or later (Tanzu Kubernetes Grid Service) - note very specific to VKS version 3.3.0, some commands may fail in newer versions.
- vSphere Namespace access with at least edit permissions
- Jumpbox with jq and kubectl, kubectl-vsphere plugin installed

## Files
- svc-account-issuer-custom-class-3.3.0.yaml   - This is the custom cluster class that exposes the service-account-issuer variable
- test-svc-cluster-3.3.0.yaml - VKS cluster manifest with cluster secret that leverage the custom cluster class
- patch-issuer.yaml - json patch file that will update the service-account-issuer value

## Proccedure 
**Note:** All kubectl commands are run from the Supervisor context

1. Modify namepace in svc-account-issuer-custom-class-3.3.0.yaml cluster class manifest with the vSphee namespace where you intend to create your vks cluster
2. Create the custom cluster class
```
kubectl apply -f svc-account-issuer-custom-class-3.3.0.yaml 
```
3. Modify test-svc-cluster-3.3.0.yaml with the correct values for the following:

**Secret** section
- name: Must be in form of <clustername>-sa (test-svc-cluster-330-sa in our example)
- namespace: vSphere namespace where the VKS cluster will be created (test-ns in our example)

**Cluster** section
- name: <clustername> (test-svc-cluster-330 in our example)
- namespace: vSphere namespace where VKS cluster will be created (test-ns in our example)
- class: names of the cluster class we are using (svc-account-issuer-custom-class-3.3.0 in our example)
- vmClass: must match an available vmclass assigned to your vsphere namespace (best-effort-medium in our example).  Use command below to determine
- storageClass: must match an avaiable storageclass for your vsphere namespace (vsan-default-storage-policy in our example).  Use command below to determine.
- defaultStorageClass: set to same value as storageClass

vmClass spec.topology.variables[name="vmClass"].value
```
kubectl get vmclass -n test-ns
```

storageClass spec.topology.variables[name="storageClass"].value
```
kubectl describe ns test-ns
```

Example Output: storage class is vsan-default-storage-policy
```
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
kubectl get cluster,kcp,md,ma,vspheremachine -n test-ns
```
6. Check Default Service Account Issuer Value
```
kubectl get kcp -n test-ns
```
```
kubectl get kcp test-svc-cluster-330-xxxxx -n test-ns -o json | jq -r '.spec.kubeadmConfigSpec.clusterConfiguration.apiServer.extraArgs["service-account-issuer"]'
```
Output should display
https://kubernetes.default.svc.cluster.local


## Patching
One the OIDC endpoint has been created we can patch the cluster with the new value.  You have 2 methods to chose from but I'd recommed the Direct Patch method.  
**Note:** Either patch method will trigger a rolling update of your control-plane node(s).

### Merge Patch
A merge type patch is easiest because you don't need to determine the index of the variable you wish to patch.  However, a merge patch will be denied by the validating webhook unless you supply all values under the variables section of the manifest.  Here is an example of a merge patch based on our simple VKS cluster manifest.  Note: you would need to update the vmclass, storageClass and defaultStorageClass values to match your environment.
```
kubectl patch cluster test-svc-cluster-330 -n test-ns --type merge -p '
{
  "spec": {
    "topology": {
      "variables": [
        {
          "name": "serviceAccountIssuer",
          "value": "https://login.microsoftonline.com/tenant123/v2.0"
        },
        {
          "name": "vmClass",
          "value": "best-effort-medium"
        },
        {
          "name": "storageClass",
          "value": "vsan-default-storage-policy"
        },
        {
          "name": "vsphereOptions",
          "value": {
            "persistentVolumes": {
              "defaultStorageClass": "vsan-default-storage-policy"
            }
          }
        }
      ]
    }
  }
}'
```
### Direct Variable Patch (Recommended)
If you have a more complex VKS manifest the merge patch can become quite complex with many variables to manage.  It also introduces the potential to break the cluster if variables are incorrect.  This direct patch 2-step method is safer.

1. Determine Index Location
```
kubectl get cluster test-svc-cluster-330 -n test-ns -o jsonpath='{range .spec.topology.variables[*]}{.name}{"\n"}{end}' | grep -n "serviceAccountIssuer"
```
This will return something like the following:
```
1:serviceAccountIssuer
```
We need to correct the index number (x-1) becaue Grep starts at 1 and Kubernetes starts at 0.  So index 1 becomes 0

2. Create a Patch File (or update patch-issuer.yaml in this repository)
```
- op: replace
  path: /spec/topology/variables/0/value                             # Update the index for correct value
  value: "https://login.microsoftonline.com/tenant123/v2.0"          # update with OIDC endpoint (no terminal /)
```

3. Patch Cluster
```
 k patch cluster test-svc-cluster-330 \
 -n test-ns \
 --type json \
 --patch-file patch-issuer.yaml
 ```

 Note: This will trigger a rolling update of the control plane node(s) in the cluster

## Verification

1. Get kcp object name
```
kubectl get kcp -n test-ns 
```
2. Veriy Service-Account-Issuer is Updated
```
kubectl get kcp test-svc-cluster-330-xxxxx -n test-ns -o json | jq -r '.spec.kubeadmConfigSpec.clusterConfiguration.apiServer.extraArgs["service-account-issuer"]'
```
Output Should Show Value from Patch

3. Check Well-Known ID for Cluster (from VKS cluster context)
```
 kubectl get --raw /.well-known/openid-configuration |jq
```

