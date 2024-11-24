# CSI Dricer

# add the CSI Driver to the helm

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault --set "server.dev.enabled=true" --set "injector.enabled=false" --set "csi.enabled=true"
```

# Csi  Driver in the cluster
```bash 
kubectl exec -it vault-0 -- /bin/sh
```

# Create a secret at the path secret/db-pass with a password.

```bash
vault kv put secret/db-pass password="db-secret-password"
```

# Verify that the secret is readable at the path secret/db-pass.

```bash
vault kv get secret/db-pass
```

# Configure Kubernetes authentication

```bash
vault auth enable kubernetes
```    

# Enable Kubernetes auth in Vault using its pod's service account token.

```bash
vault write auth/kubernetes/config kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```


# policy internal-app
```bash
vault policy write internal-app - <<EOF
path "secret/data/db-pass" {
  capabilities = ["read"]
}
EOF

```

# create a Kubernetes authentication role named database that binds this policy with a Kubernetes service account named

```bash
vault write auth/kubernetes/role/database bound_service_account_names=webapp-sa bound_service_account_namespaces=default policies=internal-app ttl=20m
```

# exit 
```bash
exit
```


# Install CSI Drive secrets 
```bash 
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi secrets-store-csi-driver/secrets-store-csi-driver  --set syncSecret.enabled=true

```

# write the secret provider class yaml

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-database
spec:
  provider: vault
  parameters:
    vaultAddress: "http://vault.default:8200"
    roleName: "database"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/db-pass"
        secretKey: "password"
```

# create a Kubernetes service account named webapp-sa
```bash
kubectl create serviceaccount webapp-sa
```

# install the webapp 

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: webapp
spec:
  serviceAccountName: webapp-sa
  containers:
  - image: jweissig/app:0.0.1
    name: webapp
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets-store"
      readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "vault-database"
```


