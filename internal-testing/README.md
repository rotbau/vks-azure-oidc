# Internal VKS Testing

Because my testing environment is not integrated with Azure AD Workload Idenity I needed to validate the SA key pair update without the back-end Azure OIDC components.  This section documents validating the process using the default VKS service account issuer `--service-account-issuer"https://kubernetes.default.svc.cluster.local"`

1. Create the VKS custom cluster class and test-svc-cluster-330 using the manifests provided in the [intial setup](../README.md)
2. Skip the patch-issuer process.  This will result in a cluster using the default `--service-account-issuer`
3. Follow the service-account-key-rotation process for VKS [service-account-key-rotation section](../service-account-key-rotation/README.md) skipping any of the Azure back-end sections.
4. For Section 5 (Validation) of the upstream Azure AD Workload Idenity documentation - deploy the `dummy-pod-vks.yaml manifest` provided in this repository.  This deploys a pod with the updated SA key pair.
5. Validate POD is using the new SA key pair.
```
kubectl exec dummy-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | cut -d. -f1 | base64 -d
{"alg":"RS256","kid":"kTDGXXd8VyqQDvF7fWqdyqDFOVVriZRESel9PaekVZg"}
```
```
openssl pkey -pubin -in sa-new.pub -outform DER | openssl dgst -sha256 -binary | base64 | tr '+/' '-_' | tr -d '='
kTDGXXd8VyqQDvF7fWqdyqDFOVVriZRESel9PaekVZg
```
