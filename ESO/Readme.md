# External Secrets Operator (ESO)

Your setup process for **External Secrets Operator (ESO)** looks good, and it should work with a few adjustments to make sure everything is properly configured.

---

# Install External Secrets Operator using Helm

```bash
helm repo add external-secrets https://charts.external-secrets.io

helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace 

```
---
# Create a Secret Containing Your AWS Credentials
```bash
kubectl create secret generic awssm-secret --from-file=./access-key --from-file=./secret-access-key
```
---
# Create a Secret Store for AWS Credentials


```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: awssm-secret
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: awssm-secret
            key: access-key
          secretAccessKeySecretRef:
            name: awssm-secret
            key: secret-access-key

```

Apply the configuration:

```bash
kubectl apply -f Secretstore.yaml
```
---
# Create External Secret to Fetch Secret from AWS Secrets Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: demo
spec:
  refreshInterval: 10m
  secretStoreRef:
    name: awssm-secret
    kind: SecretStore
  target:
    name: kube-secret
    creationPolicy: Owner
  dataFrom:
    - extract:
        key: my_DB

```


```bash 
kubectl apply -f External-secret.yaml
```
---

# Verify the Secrets in Kubernetes

```bash
kubectl get secret kube-secret -o yaml
```