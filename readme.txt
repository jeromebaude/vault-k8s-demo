DEPLOY DEMO

$ kubectl create namespace namespace-demo
$ kubectl config set-context --current --namespace=namespace-demo

$ vault namespace create namespace-demo
$ export VAULT_NAMESPACE=namespace-demo

$ vault auth enable kubernetes
$ vault auth enable userpass

$ kubectl create -f app-service-account.yml

$ TOKEN_REVIEW_JWT=$(kubectl get secret vault-auth -o go-template='{{ .data.token }}' | base64 --decode)

$ KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 --decode)

$ KUBE_HOST=https://kubernetes.docker.internal:6443

# Config of k8s Auth Method
$ vault write auth/kubernetes/config token_reviewer_jwt="$TOKEN_REVIEW_JWT" kubernetes_host="$KUBE_HOST" kubernetes_ca_cert="$KUBE_CA_CERT"

# Enable kv secret engine
$ vault secrets enable -version=1 -path=my-secrets kv

# create a secret
$ vault kv put my-secrets/my-secret username=jerome password=Passw0rd ttl=30s

# deploy a policy
$ vault policy write policy-nsdemo policy-nsdemo.hcl

# Configure the k8s role
$ vault write auth/kubernetes/role/myrole bound_service_account_names=vault-auth bound_service_account_namespaces=namespace-demo policies=policy-nsdemo ttl=1h

# deploy the app
$ kubectl create -f app.yaml

# see the deployed app GUI
$ kubectl port-forward pod/app-76ff7dc548-62mwp 8080:80

# enable vault agent injection with external vault server address
$ cd ./vault-helm
$ helm install vault --set "injector.externalVaultAddr=http://192.168.0.12:8200" .

$ kubectl patch deployment app --patch "$(cat app-patch.yaml)"
###############PREPARATION BY AN ADMIN###################
# Create a service account, 'app'
$ kubectl create serviceaccount app

# Update the 'app' service account
$ kubectl apply -f  app-service-account.yml

# Create a policy named app
$ vault policy write -namespace=namespace_demo app policy_nsdemo.hcl

# Enable k8s Auth Method on Vault
$ vault auth enable -namespace=namespace_demo  kubernetes

# Prepare variable to configure k8s Auth Method
$ export VAULT_SA_NAME=$(kubectl get sa app -o jsonpath="{.secrets[*]['name']}" -n namespace_demo)
$ export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" -n namespace_demo | base64 --decode; echo)
$ export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" -n namespace_demo | base64 --decode; echo)

# Configure the k8s Auth Method
$ vault write auth/kubernetes/config -namespace=namespace_demo token_reviewer_jwt="$SA_JWT_TOKEN" kubernetes_host=https://kubernetes.docker.internal:6443 kubernetes_ca_cert="$SA_CA_CRT"

# Configure the k8s role
$ vault write auth/kubernetes/role/myrole -namespace=namespace_demo bound_service_account_names=app bound_service_account_namespaces=namespace_demo policies=policy_nsdemo ttl=1h

# deploy the app
$ kubectl patch deployment app --patch "$(cat app-patch.yaml)"


# link
$ kubectl exec -it pod/  -c nginx
$ ln -sfF /vault/secrets/index.html /usr/share/nginx/html/index.html

ACCESS
$ kubectl port-forward pod/app-5d878b47d4-th582  8080:80

#UPDATE
#$ kubectl patch deployment app --patch "$(cat patch.yaml)"

DELETE
$ kubectl delete -f app3.yaml

DEBUG
$ kubectl logs pod/app-6675fc8cc4-j4whf -c vault-agent-init
