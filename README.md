# VKS Cluster Class and Cluster for Custom Service Account Provider

These files can be used to add a custom OIDC endpoint for service account authorization and role assumption on VKS clusters

## Files
- svc-account-issuer-custom-class-3.4.0.yaml   - This is the custom cluster class that exposes the service-account-issuer variable
- test-svc-cluster.yaml - VKS cluster manifest with cluster secret that leverage the custom cluster class
- patch-issuer.yaml - json patch file that will update the service-account-issuer value

## Patching
A merge type patch fails because the cluster validating webhook expects all values under the .spec.topology.variables to be provided in the patch.  This is undesirable and may cause the entire cluster to re-roll nodes instead of just the control plane nodes.  To get around this we use a 2 part process.  We determine the correct index of the service-account-issuer variable and then patch it directly.

Determine Index Location
```
kubectl get cluster test-svc-cluster -n test-ns -o jsonpath='{range .spec.topology.variables[*]}{.name}{"\n"}{end}' | grep -n "serviceAccountIssuer"
1:serviceAccountIssuer
```
We need to correct the index number (x-1) becaue Grep starts at 1 and Kubernetes starts at 0.  So index 1 becomes 0

Now we create a patch file
```
- op: replace
  path: /spec/topology/variables/0/value                             # Update the index for correct value
  value: "https://login.microsoftonline.com/tenant123/v2.0"          # update with OIDC endpoint (no terminal /)
```

Then Patch
```
 k patch cluster test-svc-cluster \
 -n test-ns \
 --type json \
 --patch-file patch-issuer.yaml
 ```

Check the update applied
```
kubectl get kcp test-svc-cluster-7vtmt -n test-ns -o json | jq -r '
  .spec.kubeadmConfigSpec.clusterConfiguration.apiServer.extraArgs[] 
  | select(.name == "service-account-issuer") 
  | .value'
```

Check Well-Known ID for Cluster
```
 kubectl get --raw /.well-known/openid-configuration |jq
```

