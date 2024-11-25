# Sealed Secrets

![architecture](<Screenshot from 2024-11-23 15-05-11.png>)

# **Managing Secrets with Sealed Secrets in Kubernetes**

This guide explains how to securely manage Kubernetes secrets using Sealed Secrets. It includes steps for installation, creating, encrypting, and decrypting secrets, as well as applying them to a cluster.

---
## Install sealed-secrets
Add the Sealed Secrets Helm repository and install the Sealed Secrets controller in the `kube-system` namespace:

```bash 
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install my-release sealed-secrets/sealed-secrets -n sealed-secrets --create-namespace
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

# name of the --controller-name

```bash
kubectl get svc -n sealed-secrets
```
---

# connect to kubeseal with cluster

```bash
kubeseal --fetch-cert --controller-name my-release-sealed-secrets  --controller-namespace sealed-secrets
```
---

# create the sealed secret 

```bash
kubectl create secret generic database -n default --from-literal=DB_PASSWORD=enterthepassword --dry-run=client -o yaml > secert.yaml 
```
---

# the secret.yaml secret is encoded 

```yaml
apiVersion: v1
data:
  DB_PASSWORD: ZW50ZXJ0aGVwYXNzd29yZA==
kind: Secret
metadata:
  creationTimestamp: null
  name: database
  namespace: default
```

----

# Decode the Base64-Encoded Secret (Optional)
```bash
echo 'ZW50ZXJ0aGVwYXNzd29yZA==' | base64 --decode
```
---

# apply the secret in the cluster
```bash
kubectl apply -f secert.yaml
```
---

#  Encrypt the Secret with Sealed Secrets

```bash
kubeseal -o yaml --controller-name my-release-sealed-secrets  --controller-namespace sealed-secrets < secert.yaml > sealed-secert.yaml 
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

# create the POD to mouth the secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
  labels:
    name: demo
spec:
  containers:
  - name: demo-pod
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/secrets"
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: database


```

```bash
kubectl apply -f pod.yaml
```

# verify the secret
```bash
kubectl exec -it <pod-name> -- /bin/sh
cat /etc/secrets/DB_PASSWORD
```

## **Summary**
- Sealed Secrets ensures the secure management of secrets by encrypting them before storing in version control.
- Only the Sealed Secrets controller in your cluster can decrypt the secrets.
- The `kubeseal` CLI tool is used to encrypt secrets.
- The `kubectl` command is used to apply the sealed secret to the cluster.


## **Benefits of Sealed Secrets**

1. **Secure Storage in Version Control**:  
   Secrets are encrypted before being committed to Git, enabling secure GitOps workflows.

2. **Encryption at Rest**:  
   Secrets are protected by the Sealed Secrets controller's private key, ensuring strong encryption.

3. **Kubernetes Native Integration**:  
   Seamlessly integrates with Kubernetes resources, creating standard `Secret` objects after decryption.

4. **Ease of Use**:  
   Simple deployment with Helm charts and user-friendly encryption/decryption using the `kubeseal` CLI tool.

5. **Auditability**:  
   Encrypted secrets stored in version control provide a clear history of changes for audit purposes.

6. **Cluster-Specific Encryption**:  
   Secrets are bound to a specific cluster, enhancing security by making them unusable outside the intended environment.

7. **Key Rotation Support**:  
   Automatic re-encryption of secrets when private keys are rotated, ensuring secure lifecycle management.

---

## **Limitations of Sealed Secrets**

1. **Controller Dependency**:  
   Requires the Sealed Secrets controller to be running for decrypting secrets. If the controller is unavailable, secrets cannot be accessed.

2. **Single-Cluster Usability**:  
   Encrypted secrets are cluster-specific and cannot be reused across clusters without re-encryption.

3. **No Dynamic Secret Management**:  
   Unlike tools like HashiCorp Vault, it does not support dynamic generation of secrets such as database credentials.

4. **Limited Access Control**:  
   Relies on Kubernetes RBAC for access management, lacking advanced policy control seen in dedicated secret management tools.

5. **Risk of Key Loss**:  
   Losing the controller's private key renders encrypted secrets unusable, making backups of the key critical.
