# Deploy and Use Virtual-Secrets

## Install KubeVault
```bash
helm install kubevault oci://ghcr.io/appscode-charts/kubevault \
  --version v2025.2.10 \
  --namespace kubevault --create-namespace \
  --set-file global.license=/path/to/the/license.txt \
  --wait --burst-limit=10000 --debug
```

## Deploy VaultServe
```yaml
apiVersion: kubevault.com/v1alpha2
kind: VaultServer
metadata:
  name: vault
  namespace: demo
spec:
  allowedSecretEngines:
    namespaces:
      from: All
  version: 1.12.1
  replicas: 3
  backend:
    raft:
      storage:
        storageClassName: "standard"
        resources:
          requests:
            storage: 1Gi
  unsealer:
    secretShares: 5
    secretThreshold: 3
    mode:
      kubernetesSecret:
        secretName: vault-keys
```
```bash
$ kubectl apply -f manifests/vault-server.yaml
```

## Provide permission to Virtual-Secrets Server
We will connect to the Vault by using Vault CLI. Therefore, we need to export the necessary environment variables and port-forward the service.
```bash
$ export VAULT_ADDR=http://127.0.0.1:8200
$ export VAULT_TOKEN=(kubectl vault root-token get vaultserver vault -n demo --value-only)
```

Now, Let’s port-forward the service and interact via CLI,
```bash
$ kubectl port-forward -n demo service/vault 8200
Forwarding from 127.0.0.1:8200 -> 8200
Forwarding from [::1]:8200 -> 8200

$ vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.12.1
Build Date      2022-10-27T12:32:05Z
Storage Type    s3
Cluster Name    vault-cluster-3bd6b372
Cluster ID      15df69fb-e717-9af1-8d00-0f4cc9df97d4
HA Enabled      false
```

Now, enable kv secret engine in the path virtual-secrets.dev where the server will store the secrets:
```bash
$ vault secrets enable -path=virtual-secrets.dev -version=2 kv
```
Then, provide permission to virtual-secrets server to use this secret-engine:
```bash
$ vault policy write virtual-secrets-policy server-policy.hcl
$ vault write auth/kubernetes/role/virtual-secrets-role \
    bound_service_account_names=virtual-secrets-server \
    bound_service_account_namespaces=kubevault \
    policies="virtual-secrets-policy"
```

## Create SecretSource
```yaml
apiVersion: config.virtual-secrets.dev/v1alpha1
kind: SecretSource
metadata:
  name: vault
spec:
  vault:
    url: http://vault.demo:8200
    roleName: virtual-secrets-role
```
```bash
$ kubectl apply -f manifests/secret-source.yaml
```

## Create Virtual-Secret:
```yaml
apiVersion: virtual-secrets.dev/v1alpha1
kind: Secret
metadata:
  name: virtual-secret
  namespace: demo
stringData:
  username: appscode
  password: password
secretSourceName: vault
```
```bash
$ kubectl apply -f manifests/virtual-secret.yaml
```

## Deploy Postgres with Virtual-Secret:
```bash
apiVersion: kubedb.com/v1
kind: Postgres
metadata:
  name: pg-demo
  namespace: demo
spec:
  authSecret:
    apiGroup: "virtual-secrets.dev"
    secretSourceName: vault
  replicas: 3
  storage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
    storageClassName: standard
  storageType: Durable
  deletionPolicy: Delete
  version: "13.13"
```
```bash
$ kubectl apply -f manifests/postgres.yaml
```