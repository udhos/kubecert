# kubecert

Source: https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/

```
kind --version
kind version 0.17.0

kubectl version
WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.6", GitCommit:"ff2c119726cc1f8926fb0585c74b25921e866a28", GitTreeState:"clean", BuildDate:"2023-01-18T19:22:09Z", GoVersion:"go1.19.5", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.7
Server Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.3", GitCommit:"434bfd82814af038ad94d62ebe59b133fcb50506", GitTreeState:"clean", BuildDate:"2022-10-25T19:35:11Z", GoVersion:"go1.19.2", Compiler:"gc", Platform:"linux/amd64"}

kind create cluster --name cert

user=john

openssl genrsa -out $user.key 2048
openssl req -new -key $user.key -out $user.csr

# CN is the name of the user
# O is the group that this user will belong to

request=$(cat $user.csr | base64 | tr -d "\n")

kubectl apply -f - <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: $user
spec:
  request: $request
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF

kubectl get csr

kubectl certificate approve $user

kubectl get csr/$user -o yaml

kubectl get csr $user -o jsonpath='{.status.certificate}'| base64 -d > $user.signed.crt

#
# create role for user
#

kubectl create role developer --verb=create --verb=get --verb=list --verb=update --verb=delete --resource=pods,pods/log

kubectl create rolebinding developer-$user --role=developer --user=$user

#
# kubeconfig
#

kubectl config set-credentials $user --client-key=$user.key --client-certificate=$user.signed.crt --embed-certs=true

kubectl config set-context $user --cluster=kind-cert --user=$user

kubectl config use-context $user

kubectl get po

kubectl run busybox --image busybox --command echo hi

kubectl logs busybox

#
# Destroy everything
#

kubectl config use-context kind-cert

kind delete cluster --name cert
```
