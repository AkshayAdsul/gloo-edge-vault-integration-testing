# Gloo Edge Vault Integration Testing

This project is for testing Gloo Edge `1.14.x` integration with Vault (as a general K/V secret store) and as a CA for issuing certificates. Certificates will be issued by cert-manager using Vault as the CA. Generated certs will be stored as Kubernetes secrets.
Uses the [base bootstrapping project](https://github.com/pseudonator/gloo-edge-1-14) to deploy the EKS infra and Gloo Edge.

## Deployment

1. Deploy GE bootstrapping project

    ```
    mkdir -p ._output
    git clone https://github.com/pseudonator/gloo-edge-1-14 ._output/gloo-edge-1-14

    export CLUSTER_OWNER="kasunt"
    export PROJECT="gloo-ee-vault-integration"

    export CLOUD_PROVIDER="eks"
    export EKS_CLUSTER_REGION=ap-southeast-2

    export DOMAIN_NAME=testing.development.internal

    export GLOO_EDGE_HELM_VERSION=1.14.1
    export GLOO_EDGE_VERSION=v${GLOO_EDGE_HELM_VERSION}

    export CERT_MANAGER_VERSION="v1.11.2"
    export VAULT_VERSION="0.24.1"

    ._output/gloo-edge-1-14/cluster-provision/scripts/provision-eks-cluster.sh create -n $PROJECT -o $CLUSTER_OWNER -a 3 -v 1.25 -r $EKS_CLUSTER_REGION

    ._output/gloo-edge-1-14/setup.sh -f gloo-edge-override-helm-values.yaml -i
    ```

2. Install Sample App

    ```
    kubectl create ns apps

    kubectl apply -f apps/deploy-petstore.yaml
    ```

3. Configuration

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

    Application configuration

    ```
    kubectl create ns apps-configuration

    kubectl apply -f configuration
    ```

## Clean up

```
._output/gloo-edge-1-14/cleanup.sh

._output/gloo-edge-1-14/cluster-provision/scripts/provision-eks-cluster.sh delete -n $PROJECT -o $CLUSTER_OWNER -r $EKS_CLUSTER_REGION
```

## Testing

```
curl -kiv -H "Host: cert.test.gloo" https://$(kubectl get svc gateway-proxy -n gloo-system -o=jsonpath='{.status.loadBalancer.ingress[0].hostname}')/get-pets
```