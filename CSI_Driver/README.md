This setup will allow you to securely access Vault secrets using the Secrets Store CSI driver in Kubernetes. Below is a well-structured guide to configure and deploy the Vault integration using the CSI driver.

### 1. **Add the HashiCorp Helm Repository**
   To add the HashiCorp Helm repository and install Vault, use the following command:
   ```bash
   helm repo add hashicorp https://helm.releases.hashicorp.com
   helm install vault hashicorp/vault --set "server.enabled=false" --set "injector.enabled=false" --set "csi.enabled=true"
   ```

### 2. **Check the Pods**
   Once Vault is deployed, check the running pods with:
   ```bash
   kubectl get pods
   ```

### 3. **Set a Secret in Vault**
   Enter the Vault pod and create a secret:
   ```bash
   kubectl exec -it vault-0 -- /bin/sh
   vault kv put secret/db-pass password="db-secret-password"
   ```

### 4. **Verify the Secret in Vault**
   To ensure the secret was stored successfully, run:
   ```bash
   vault kv get secret/db-pass
   ```

### 5. **Enable Kubernetes Authentication in Vault**
   Enable the Kubernetes authentication method:
   ```bash
   vault auth enable kubernetes
   ```

   Configure the Kubernetes authentication:
   ```bash
   vault write auth/kubernetes/config kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
   ```

### 6. **Create a Vault Policy for Internal Application**
   Define a policy to allow access to the secret path:
   ```bash
   vault policy write internal-app - <<EOF
   path "secret/data/db-pass" {
     capabilities = ["read"]
   }
   EOF
   ```

### 7. **Create a Kubernetes Authentication Role**
   Bind the Kubernetes service account to the Vault policy:
   ```bash
   vault write auth/kubernetes/role/database \
       bound_service_account_names=webapp-sa \
       bound_service_account_namespaces=default \
       policies=internal-app \
       ttl=20m
   ```

### 8. **Exit the Vault Pod**
   Once done, exit the Vault pod:
   ```bash
   exit
   ```

### 9. **Install the Secrets Store CSI Driver**
   Add the Secrets Store CSI driver repository and install it:
   ```bash
   helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
   helm install csi secrets-store-csi-driver/secrets-store-csi-driver --set syncSecret.enabled=true
   ```

### 10. **Verify the Vault CSI Provider is Running**
   Ensure the CSI provider is running by checking the pods:
   ```bash
   kubectl get pods
   ```

### 11. **Create a SecretProviderClass Resource**
   Define the `SecretProviderClass` for the Vault secrets provider:
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

### 12. **Apply the SecretProviderClass Resource**
   Apply the `SecretProviderClass` resource:
   ```bash
   kubectl apply -f SecretProviderClass.yaml
   ```

### 13. **Create a Kubernetes Service Account**
   Create the service account for your web application:
   ```bash
   kubectl create serviceaccount webapp-sa
   ```

### 14. **Create the WebApp Pod for Mounting the Secret**
   Define the pod that will mount the secret from the Vault:
   ```yaml
   kind: Pod
   apiVersion: v1
   metadata:
     name: webapp
   spec:
     serviceAccountName: webapp-sa
     containers:
     - image: nginx
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

### 15. **Display the Mounted Secret in the Pod**
   After the pod is running, verify that the secret has been mounted correctly:
   ```bash
   kubectl exec webapp -- cat /mnt/secrets-store/db-password
   ```

### 16. **Sync the Secret to a Kubernetes Secret**
   To sync the secret from Vault to a Kubernetes secret, use the following `SecretProviderClass` configuration:
   ```yaml
   apiVersion: secrets-store.csi.x-k8s.io/v1
   kind: SecretProviderClass
   metadata:
     name: vault-database
   spec:
     provider: vault
     secretObjects:
     - data:
       - key: password
         objectName: db-password
       secretName: dbpass
       type: Opaque
     parameters:
       vaultAddress: "http://vault.default:8200"
       roleName: "database"
       objects: |
         - objectName: "db-password"
           secretPath: "secret/data/db-pass"
           secretKey: "password"
   ```

### 17. **Update the SecretProviderClass for Secret Syncing**
   If you need to sync the secret with a Kubernetes secret, update the `SecretProviderClass` as shown below:
   ```yaml
   apiVersion: secrets-store.csi.x-k8s.io/v1
   kind: SecretProviderClass
   metadata:
     name: vault-database
   spec:
     provider: vault
     secretObjects:
     - data:
       - key: password
         objectName: db-password
       secretName: dbpass
       type: Opaque
     parameters:
       vaultAddress: "http://vault.default:8200"
       roleName: "database"
       objects: |
         - objectName: "db-password"
           secretPath: "secret/data/db-pass"
           secretKey: "password"
   ```

By following these steps, you will successfully integrate Vault with the Secrets Store CSI Driver in Kubernetes to securely manage and mount secrets in your pods.