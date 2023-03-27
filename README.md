# Hashicorp Vault setup for COSIGN

## Install the bits and go 

0. Add Helm repo for hashicorpt
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com && helm repo update
```

1. Deploy the chart 
```
helm upgrade vault hashicorp/vault -i --create-namespace -n vault --atomic -f $PWD/values.yaml
```

2. Unseal / seal the vault (stolen from [here](https://github.com/redhat-cop/vault-config-operator/blob/main/docs/end-to-end-example.md))
```bash 
INIT_RESPONSE=$(oc exec vault-0 -n vault -- vault operator init -address https://vault.vault.svc:8200 -ca-path /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt -format=json -key-shares 1 -key-threshold 1)

UNSEAL_KEY=$(echo "$INIT_RESPONSE" | jq -r '.unseal_keys_b64[0]')
ROOT_TOKEN=$(echo "$INIT_RESPONSE" | jq -r .root_token)

echo "$UNSEAL_KEY"
echo "$ROOT_TOKEN"

#here we are saving these variable in a secret, this is probably not what you should do in a production environment
oc delete secret vault-init -n vault
oc create secret generic vault-init -n vault --from-literal=unseal_key=${UNSEAL_KEY} --from-literal=root_token=${ROOT_TOKEN}
export UNSEAL_KEY=$(oc get secret vault-init -n vault -o jsonpath='{.data.unseal_key}' | base64 -d )
export ROOT_TOKEN=$(oc get secret vault-init -n vault -o jsonpath='{.data.root_token}' | base64 -d )
oc exec vault-0 -n vault -- vault operator unseal -address https://vault.vault.svc:8200 -ca-path /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt $UNSEAL_KEY
```


3. Setup kubernetes auth stuffs (not sure if this is actually needed 🤷)
```bash
oc exec -ti vault-0 -- sh -c "JWT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) && KUBERNETES_HOST=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 && vault auth enable --tls-skip-verify kubernetes"
```

4. Enable transits secret engine (KMS for vault) 
```bash
oc exec vault-0 -n vault -- vault login --tls-skip-verify $ROOT_TOKEN 
oc exec vault-0 -n vault -- vault secrets enable --tls-skip-verify transit
```

5. [Key Generations](https://github.com/sigstore/cosign/blob/6df3ad928010206929f98491171f41adbdc0d780/KMS.md#key-generation-and-management) ... assumes $VAULT_ADDR and $VAULT_TOKEN are set
```bash
export cluster_base_domain=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export VAULT_ADDR=https://vault-vault.apps.${cluster_base_domain}
export VAULT_TOKEN=$(oc get secret vault-init -n vault -o jsonpath='{.data.root_token}' | base64 -d )

cosign generate-key-pair --kms hashivault://my-super-duper-private-key
```


## clean up
```
helm uninstall vault && oc delete project vault
```
