# Sealed Secrets


# **Managing Secrets with Sealed Secrets in Kubernetes**

This guide explains how to securely manage Kubernetes secrets using Sealed Secrets. It includes steps for installation, creating, encrypting, and decrypting secrets, as well as applying them to a cluster.

---
## Install sealed-secrets
Add the Sealed Secrets Helm repository and install the Sealed Secrets controller in the `kube-system` namespace:

```bash 
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install my-release sealed-secrets/sealed-secrets -n kube-system
```
---

# Install kubeseal 

```bash
KUBESEAL_VERSION='0.27.2'
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION:?}/kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```
---

# connect to kubeseal with cluster

```bash
kubeseal --fetch-cert --controller-name my-release-sealed-secrets  --controller-namespace kube-system
```
---

# create the sealed secret 

```bash
kubectl create secret generic database -n default --from-literal=DB_PASSWORD=password_123 --dry-run=client -o yaml > secert.yaml 
```
---

# the secret.yaml secret is encoded 

```bash
apiVersion: v1
data:
  DB_PASSWORD: cGFzc3dvcmRfMTIz
kind: Secret
metadata:
  creationTimestamp: null
  name: database
  namespace: default
```

----

# Decode the Base64-Encoded Secret (Optional)
```bash
echo 'cGFzc3dvcmRfMTIz' | base64 --decode
```

---

#  Encrypt the Secret with Sealed Secrets

```bash
kubeseal -o yaml --controller-name my-release-sealed-secrets  --controller-namespace kube-system < secert.yaml > sealed-secert.yaml 
```
---

#  Apply the Sealed Secret to the Cluster

```bash
kubectl apply -f sealed-secert.yaml
```
---

# Retrieve the Original Secret

```bash 
kubectl get secret database -n default -o yaml
```

---

## **Summary**
- Sealed Secrets ensures the secure management of secrets by encrypting them before storing in version control.
- Only the Sealed Secrets controller in your cluster can decrypt the secrets.
- The `kubeseal` CLI tool is used to encrypt secrets.
- The `kubectl` command is used to apply the sealed secret to the cluster.