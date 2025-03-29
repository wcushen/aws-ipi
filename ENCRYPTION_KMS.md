# **ODF Cluster-Wide Encryption with HashiCorp Vault KMS**

## **Introduction**

Managing secrets securely is becoming increasingly a non-neogtiable for many enterprises today across cloud and on-prem environments. With IBM’s recent acquisition of HashiCorp, organizations are now poised to take advantage of deeper integration opportunities between HashiCorp Vault and Red Hat OpenShift, strengthening their security posture.

In this article, we’ll walk through the integration of OpenShift Data Foundation (ODF) with HashiCorp Vault to enable cluster-wide encryption using Vault as a Key Management System (KMS). This demo will be deployed on AWS using local devices as the backing store to the ODF cluster.

## **Deploying Vault on OpenShift**

While many enterprises use an **external Vault instance**, for demo purposes, we will deploy Vault **within the OpenShift cluster** using Helm. The Vault instance will:

- Be deployed via **Helm**.
- Use TLS certificates managed by **cert-manager** (not covered here).
- Be initialized with a **root CA generated locally**

### **1. Deploy Vault with Helm**

First, add the HashiCorp Helm repository and install Vault:

```sh
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault --namespace vault --create-namespace \
  --set server.dev.enabled=true
```

### **2. Initialize Vault**

Once Vault is running, initialize it:

```sh
vault operator init
```
This command generates **unseal keys** and a **root token**. Make sure to store them securely. You'll often be prompted to unseal Vault.

Unseal Vault with:
```sh
vault operator unseal
```

### **3. Enable Kubernetes Authentication**

```sh
vault auth enable kubernetes
```

### **4. Enable the backend path in Vault (assuming v2)**

```sh
vault secrets enable -path=odf kv-v2
```

Set up the authentication configuration:

Note: Part of the following command is used to obtain the OCP CA certificate for Vault, specifically when the OpenShift cluster is using a self-signed API certificate, and we only need the Root CA.

```sh
vault write auth/kubernetes/config \
    kubernetes_host="$(oc config view --minify --flatten -o jsonpath="{.clusters[0].cluster.server})" \
    kubernetes_ca_cert="$(openssl s_client -connect api.cushen-demo.sandbox763.opentlc.com:6443 -showcerts </dev/null 2>/dev/null | awk 'BEGIN {c=0} /-----BEGIN CERTIFICATE-----/ {c++} c==2 && NF {print} /-----END CERTIFICATE-----/ {if (c==2) exit}')"
``````

### **4. Vault ACL Policies and Roles**

#### **Create a Vault Policy for ODF**
```sh
echo '
path "odf/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "sys/mounts" {
capabilities = ["read"]
}'| vault policy write odf -
```

## ODF Vault roles 

The official ODF documentation suggests creating a dedicated `odf-vault-auth` service account in the `openshift-storage` namespace, which is assigned an auth delegator role. The auth delegator role in Vault is responsible for authenticating against the OpenShift TokenReview API. It allows Vault to verify the validity of the service account's token - so effectively, it's acting as a bridge between OpenShift's token system and Vault.

Why would we want to do this?

- Keeps the auth process separate from the ODF operator's service accounts, so we can manage access control independently.
- We can manage Vault authentication independently of the lifecycle of ODF operator components

However, for this demo, we've chosen to eliminate this redundancy. Instead of using a dedicated SA for Vault authentication, we've directly bound the **auth-delegator** role to the service account used by the ODF operator pods (`rook-ceph-system`). The `odf-rook-ceph-op` role in Vault is linked to this SA, which manages the creation of OSD keys in Vault, simplifying the configuration in our setup.

#### **Create a Role for ODF Storage**
```sh
vault write auth/kubernetes/role/odf-rook-ceph-op \
bound_service_account_names=rook-ceph-system,rook-ceph-osd,noobaa \
bound_service_account_namespaces=openshift-storage \
policies=odf \
ttl=1440h
```

```sh
vault write auth/kubernetes/role/odf-rook-ceph-osd \
bound_service_account_names=rook-ceph-osd \
bound_service_account_namespaces=openshift-storage \
policies=odf \
ttl=1440h
```
---

## **Deploy ODF StorageSystem with KMS Cluster-Wide Encryption**

Get to the stage where you've installed both the Local Storage Operator and the OpenShift Data Foundation Operator.

### **1. Create a KMS Connection Secret**

ODF requires a **KMS connection configmap** in the `openshift-storage` namespace. Create it with:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: csi-kms-connection-details
  namespace: openshift-storage
data:
  odf: '{"encryptionKMSType":"vaulttenantsa","vaultBackend":"kv-v2","kmsServiceName":"odf","vaultAddress":"https://vault.apps.cushen-demo.sandbox763.opentlc.com:443","vaultBackendPath":"odf","vaultCAFromSecret":"csi-kms-ca-secret-govhe5h8","vaultCAFileName":"my-root-ca.pem","vaultAuthMethod":"kubernetes","vaultAuthPath":"kubernetes","vaultAuthNamespace":"","vaultNamespace":""}'
```

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: ocs-kms-connection-details
  namespace: openshift-storage
data:
  KMS_SERVICE_NAME: odf
  VAULT_AUTH_KUBERNETES_ROLE: odf-rook-ceph-op
  VAULT_BACKEND: kv-v2
  VAULT_AUTH_METHOD: kubernetes
  VAULT_NAMESPACE: ''
  VAULT_CACERT: ocs-kms-ca-secret-u3e3ychq
  VAULT_AUTH_MOUNT_PATH: kubernetes
  VAULT_TLS_SERVER_NAME: ''
  VAULT_BACKEND_PATH: odf
  VAULT_ADDR: 'https://vault.apps.cushen-demo.sandbox763.opentlc.com:443'
  KMS_PROVIDER: vault
```

### **3. Restart ODF Components**

After updating the ConfigMaps, restart ODF components to apply changes:

```sh
oc delete pod -n openshift-storage --selector=app=rook-ceph-operator
```

Verify encryption is enabled:
```sh
oc get cephcluster -n openshift-storage -o yaml | grep -i encryption
```

---

## **Conclusion**

Integrating ODF with HashiCorp Vault gives you a secure, centralized way to manage encryption for your data at rest. Whether you're using Vault as your KMS or looking to enhance security with HSM-backed key storage, there's a lot of flexibility in how you approach the setup. Vault’s policies also allow for fine-grained access control, ensuring that only the right services can access your sensitive data - we can imagine a world in which certain industries require this more than others. 

This guide covers the basics, but the true value of this setup lies in its flexibility to adapt to your organization's specific needs. ODF Encryption doesn’t stop at cluster-wide encryption for OSDs; you can also configure what I would loosely reference as "double encryption". This involves encrypting not only the cluster-wide storage but also individual StorageClass-level encryption, where PVs are encrypted at the namespace level, really locking things down. 
