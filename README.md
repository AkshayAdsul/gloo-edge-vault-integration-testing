# Gloo Edge Vault Integration Testing

This project is for testing Gloo Edge `1.14.x` integration with Vault (as a general K/V secret store) and as a CA for issuing certificates. Certificates will be issued by cert-manager using Vault as the CA. Generated certs will be stored as Kubernetes secrets.
Uses the [base bootstrapping project](https://github.com/pseudonator/gloo-edge-1-14) to deploy the EKS infra and Gloo Edge.

## Deployment

1. Deploy GE bootstrapping project

    ```
    mkdir -p ._output
    git clone https://github.com/pseudonator/gloo-edge-1-14 ._output/gloo-edge-1-14

    export CLUSTER_OWNER="akshay"
    export PROJECT="gloo-ee-vault-integration"

    export CLOUD_PROVIDER="eks"
    export EKS_CLUSTER_REGION=ap-southeast-2

    export DOMAIN_NAME=testing.development.internal

    export GLOO_EDGE_HELM_VERSION=1.15.0-beta2-bcheck-multiple-secrets-api-5d4d647
    export GLOO_EDGE_VERSION=v${GLOO_EDGE_HELM_VERSION}

    export CERT_MANAGER_VERSION="v1.11.2"
    export VAULT_VERSION="0.24.1"
    
    export GLOO_EDGE_LICENSE_KEY="<value>"

    ._output/gloo-edge-1-14/cluster-provision/scripts/provision-eks-cluster.sh create -n $PROJECT -o $CLUSTER_OWNER -a 3 -v 1.25 -r $EKS_CLUSTER_REGION

    Install Gloo Edge
    helm repo add gloo-test https://storage.googleapis.com/gloo-ee-test-helm 
    helm repo update
    helm install gloo-test gloo-test/gloo -n gloo-system --version 1.15.0-beta2-bcheck-multiple-secrets-api-5d4d647 --create-namespace --set-string license_key=${GLOO_EDGE_LICENSE_KEY} -f gloo-edge-override-helm-values.yaml

    ```

2. Install Sample App

    ```
    kubectl create ns apps

    kubectl apply -f apps/deploy-petstore.yaml
    ```

3. Configuration


    Install vault on kubernetes
    Please follow https://medium.com/@tanmayvarade/hashicorp-vault-part-2-deploy-vault-on-kubernetes-edb049301d1
    
    ```
    helm repo add hashicorp https://helm.releases.hashicorp.com
    kubectl create namespace vault
    helm install vault hashicorp/vault --values vault-values.yaml -n vault
    ```
    Run the following set of commands to configure Vault to issue certs.

    ```
    kubectl port-forward po/vault-0 -n vault 8200:8200

    export VAULT_ADDR="http://127.0.0.1:8200"
    export VAULT_TOKEN="root"

    vault secrets enable pki
    vault secrets tune -max-lease-ttl=8760h pki

    vault write pki/root/generate/internal \
        common_name=test.gloo \
        ttl=8760h

    vault write pki/config/urls \
        issuing_certificates="http://vault.vault.svc:8200/v1/pki/ca" \
        crl_distribution_points="http://vault.vault.svc:8200/v1/pki/crl"

    vault write pki/roles/test-dot-com \
        allowed_domains=test.gloo \
        allow_subdomains=true \
        require_cn=false \
        max_ttl=72h

    vault policy write pki - <<EOF
    path "pki*"                     { capabilities = ["read", "list"] }
    path "pki/sign/test-dot-com"    { capabilities = ["create", "update"] }
    path "pki/issue/test-dot-com"   { capabilities = ["create"] }
    EOF
    ```


Install Cert Manager 

kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml

    Application configuration

    ```
    kubectl create ns apps-configuration

    kubectl apply -f configuration
    ```
## Make changes to Gloo Edge settings 

```
kubectl --namespace gloo-system edit settings default
```
Modify settings to remove kubernetesSecretSource: {} and  then add below at the same level the removed line was indented

```
secretOptions:
  sources:
    - vault:
        accessToken: root
        address: http://vault.vault.svc:8200
    - kubernetes: {}
``` 
If using AWS Auth you can give the same options under the vault option above as described in this link 
https://docs.solo.io/gloo-edge/latest/installation/advanced_configuration/vault_secrets/#customizing-the-gloo-edge-settings-file
## Testing

```
curl -kiv -H "Host: cert.test.gloo" https://$(kubectl get svc gateway-proxy -n gloo-system -o=jsonpath='{.status.loadBalancer.ingress[0].hostname}')/get-pets
```

## Clean up

```
._output/gloo-edge-1-14/cleanup.sh

._output/gloo-edge-1-14/cluster-provision/scripts/provision-eks-cluster.sh delete -n $PROJECT -o $CLUSTER_OWNER -r $EKS_CLUSTER_REGION
```
