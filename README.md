# 🔐 Hashicorp Vault setup for COSIGN
Setup Hashicorp as a KMS for cosign flow (extended from @sabre's demo)

## Pre req
Install the usual stuff - Helm, Cosign, Jq, oc etc. and run andy's [setup.sh script](https://github.com/sabre1041/rh-sigstore-demo) if you want to use the flow with Fulcio & Rekor

## Install the bit for Hashicorp Vault

0. Add Helm repo for hashicorpt
```bash
# oc login ...
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


3. Setup kubernetes auth stuffs (don't think this is actually needed 🤷)
```bash
# oc exec -ti vault-0 -- sh -c "JWT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) && KUBERNETES_HOST=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 && vault auth enable --tls-skip-verify kubernetes"

oc exec vault-0 -n vault -- vault auth enable --tls-skip-verify kubernetes
```

4. Enable transits secret engine (KMS for vault) 
```bash
oc exec vault-0 -n vault -- vault login --tls-skip-verify $ROOT_TOKEN 
oc exec vault-0 -n vault -- vault secrets enable --tls-skip-verify transit
```

## Signing and verifying an image
0. [Key Generations](https://github.com/sigstore/cosign/blob/main/KMS.md) ... using the `hashivault://some-key` format assumes `$VAULT_ADDR` and `$VAULT_TOKEN` are set. If you've used the setup above then this should work to get the values.
```bash
export cluster_base_domain=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export VAULT_ADDR=https://vault-vault.apps.${cluster_base_domain}
export VAULT_TOKEN=$(oc get secret vault-init -n vault -o jsonpath='{.data.root_token}' | base64 -d )

cosign generate-key-pair --kms hashivault://my-super-duper-private-key
```

1. Sign in to registry (using podman and Quay in my example)
```
podman login quay.io --authfile=$HOME/.docker/config.json -u <USER_NAME>

# inititalize cosign to use the local tuf authority
cosign initialize \
    --mirror=https://$(oc get routes -n tuf-system -o jsonpath='{.items[0].spec.host }') \
    --root=https://$(oc get routes -n tuf-system -o jsonpath='{.items[0].spec.host }')/root.json
```

2. Search and grab an image SHA
```bash
IMAGE=quay.io/petbattle/pet-battle
TAG=${IMAGE_TAG_:=latest}
podman search --list-tags ${IMAGE}
podman pull ${IMAGE}:${TAG}
export DIGEST=$(podman inspect ${IMAGE}:${TAG} | jq -r '.[0].Digest')
export KEYED_VAULT=${IMAGE}@${DIGEST}

echo ${KEYED_VAULT}
```

3. Sign the image and push to registry 
```bash
cosign sign --key hashivault://my-super-duper-private-key \
    --rekor-url=https://$(oc get routes -n rekor-system -o jsonpath='{.items[0].spec.host }') \
    ${KEYED_VAULT}
```

4. Verify
```bash
cosign verify --key cosign.pub \
    --rekor-url=https://$(oc get routes -n rekor-system -o jsonpath='{.items[0].spec.host }') \
    ${KEYED_VAULT}
```

### For a flow with no Rekor / TUF
Same setup 0 as before with generating the keys then just use these three lines:
```bash
export KEYED_VAULT=quay.io/petbattle/pet-battle:latest
cosign sign -y --key hashivault://my-super-duper-private-key $KEYED_VAULT --tlog-upload=false
cosign verify --key cosign.pub $KEYED_VAULT --insecure-ignore-tlog=true
```

## clean up
```
helm uninstall vault && oc delete project vault
```