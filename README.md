# Injecting Vault Secrets Into Kubernetes Pods via a Sidecar

This demo will showcase auto-injection of a vault-agent sidecar to manage Vault Authentication as well as Secret retrieval and templating to a Kubernetes Pod.

Our Vault instance is installed outside of Kubernetes. We are making use of the vault-k8s project (https://github.com/hashicorp/vault-k8s) for auto-injection of vault-agent.

## Let's put ourself in Dev shoes
### Simple NGINX pod deployment

    ($ kubectl config set-context --current --namespace=namespace-demo)
    $ kubectl create -f app.yaml


### Verify the right start of NGINX

    $ kubectl get pods
    $ kubectl port-forward deploy/app 8080:80


### Patch the previous deployment

    $ kubectl patch deployment app --patch "$(cat app-patch.yaml)"

### See the new pod with 2 containers

    $ kubectl get pods
    $ kubectl port-forward deploy/app 8080:80

### Let's modify the secret in Vault and check the NGINX UI
The updated secret is now displayed in NGINX since Vault-Agent checks periodically whether the secret has changed and needs to be updated

## Let's look under the cover
### Vault-Agent auto-injection

    $ kubectl get pods -n default

The auto-injection is managed by the vault-agent-injector. When targetting an external Vault, it can be installed:

    $ git clone https://github.com/hashicorp/vault-helm.git
    $ helm install vault --set "injector.externalVaultAddr=http://$VAULT_ADDR:8200"

### Exec in pod vault-agent

    $ kubectl exec -it deploy/app -c vault-agent /bin/sh

See that the vault token is located under /home/vault/.vault.token
See that secret templating is rendered in /vault/secrets

### Exec in pod nginx

    $ kubectl exec -it deploy/app -c nginx /bin/sh 

See that secrets are accessible via mount sharing of /vault/secrets
