# Previder Secure Vault and External Secrets Operator in Kubernetes

## About Previder Secure Vault

Previder Secure Vault is a secrets management service offered through the [Previder Portal](https://portal.previder.nl) self-service portal. Unlike a self-hosted secrets manager, no installation inside the Kubernetes cluster is required: a Secure Vault environment is created through the portal, and secrets are managed either through the [`vault-cli`](https://github.com/previder/vault-cli) tool or the [Secure Vault UI](https://vault.previder.io/ui). Every environment uses its own unique encryption key, which is never stored by Previder and is only available to the customer.

## About External Secrets Operator

External Secrets Operator (ESO) is a Kubernetes operator that reads secrets from an external system — in this case Previder Secure Vault — and creates a native Kubernetes `Secret` from them. Applications keep reading a normal `Secret`; ESO takes care of keeping that `Secret` in sync with what's stored in the vault. ESO has built-in, native support for Previder Secure Vault, so no custom integration is needed.

## Overview

This guide creates a Secure Vault environment through the Previder Portal, stores a secret in it with `vault-cli`, and uses External Secrets Operator's native Previder provider to deliver that secret into the cluster as a Kubernetes `Secret`.

### Token Types

Previder Secure Vault uses three token types, each with a different scope:

| | EnvironmentAdmin | ReadWrite | ReadOnly |
|---|---|---|---|
| Manage tokens | ✅ | | |
| Manage secrets | | ✅ | |
| Get decrypted secret | | ✅ | ✅ |

The token received when a Secure Vault environment is created in the portal is always an **EnvironmentAdmin** token. That token is used to create the other, more narrowly scoped tokens — it should never be used directly by an application or a cluster.

**Important:** Only a **ReadOnly** token is placed inside the Kubernetes cluster. It can read the secrets it's given the id/name of, but cannot list, create or delete secrets, and cannot create further tokens — so a compromised cluster cannot use it to gain broader access to the vault.

### Architecture Diagram

```
flowchart LR
    Portal["Previder Portal"] -->|EnvironmentAdmin token| CLI["vault-cli"]
    CLI -->|creates| RWToken["ReadWrite token"]
    CLI -->|creates| ROToken["ReadOnly token"]
    CLI -->|vault-cli secret create| Vault["Previder Secure Vault"]

    ROToken -->|stored as K8s Secret| Bootstrap["Kubernetes Secret"]
    ESO["External Secrets Operator"] -->|reads| Bootstrap
    ESO -->|Access Token auth| Vault
    Vault -->|returns secret| ESO
    ESO -->|writes| Secret["Kubernetes Secret"]
    App["Application Pod"] -->|reads| Secret
```

    Loading

---

# Prerequisites

This guide assumes:

- A running Kubernetes cluster, as described in the [Installatiehandleiding](https://github.com/previder/kubernetes-examples/tree/main/docs)
- `helm` and `kubectl` configured against the cluster
- Access to the [Previder Portal](https://portal.previder.nl) to create a Secure Vault environment
- No prior Vault or External Secrets Operator knowledge required

---

# 1. Create a Secure Vault Environment

Create a Secure Vault environment via the [Previder Portal](https://portal.previder.nl). This generates the initial **EnvironmentAdmin** token — copy it and store it in a secure location, such as a password manager. This token is only needed for the setup steps below; it is never used by the cluster itself.

---

# 2. Install vault-cli

Download the latest `vault-cli` binary for your platform from the [releases page](https://github.com/previder/vault-cli/releases/latest). For example, on Linux (amd64):

```
curl -s https://api.github.com/repos/previder/vault-cli/releases/latest \
  | grep -i "browser_download_url.*linux.*amd64" \
  | cut -d '"' -f 4 \
  | xargs curl -L -o vault-cli
```

👉 This asks the GitHub API for the latest release, finds the Linux/amd64 asset and downloads it — so the same command keeps working for future versions without needing to look up a version number by hand. For macOS or Windows, download the matching asset from the [releases page](https://github.com/previder/vault-cli/releases/latest) directly.

Make it executable and put it somewhere on your `PATH`:

```
chmod +x vault-cli
sudo mv vault-cli /usr/local/bin/vault-cli
```

Verify it works:

```
vault-cli --help
```

Set the EnvironmentAdmin token as an environment variable, so it doesn't need to be repeated on every command:

```
export VAULT_TOKEN="<environment_admin_token>"
```

---

# 3. Create a ReadOnly Token for the Cluster

The cluster only needs to **read** secrets, so it gets a ReadOnly token rather than the ReadWrite or EnvironmentAdmin token used for management:

```
vault-cli token create --description "hello-app cluster access" --type ReadOnly
```

👉 Copy the returned token — this is the value that will be stored as a Kubernetes `Secret` in step 6. Give each application (or namespace) its own ReadOnly token with its own description, rather than sharing a single token across the whole cluster, so access can be revoked per application if needed.

For managing secrets from a workstation or CI pipeline, create a separate ReadWrite token instead:

```
vault-cli token create --description "secret management from CI" --type ReadWrite
```

---

# 4. Store a Secret in the Vault

Using the ReadWrite token from step 3 (or the EnvironmentAdmin token), store an example secret — an API key for an application called `hello-app`:

```
VAULT_TOKEN="<readwrite_token>" vault-cli secret create \
  --description "hello-app-api-key" \
  --secret "s3cr3t-api-key-value"
```

List the secrets to confirm it was stored, and note its id or description — either can be used to retrieve it later:

```
VAULT_TOKEN="<readwrite_token>" vault-cli secret list -o pretty
```

---

# 5. Install External Secrets Operator

```
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

Verify:

```
kubectl -n external-secrets get pods
```

Expected output:

```
NAME                                    READY   STATUS    RESTARTS   AGE
external-secrets-...                    1/1     Running   0          30s
external-secrets-cert-controller-...    1/1     Running   0          30s
external-secrets-webhook-...            1/1     Running   0          30s
```

---

# 6. Store the ReadOnly Token as a Kubernetes Secret

Create the namespace the application will run in:

```
kubectl create namespace hello-app
```

Store the ReadOnly token created in step 3 as a Kubernetes `Secret`, directly from the command line so it never touches a file on disk:

```
kubectl -n hello-app create secret generic previder-vault-token \
  --from-literal=previder-vault-token="<readonly_token>"
```

---

# 7. Create a SecretStore

A `SecretStore` tells External Secrets Operator how to reach Previder Secure Vault. This one is scoped to the `hello-app` namespace, matching the Secret created in step 6.

`hello-app-secretstore.yaml`

```
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: previder-backend
  namespace: hello-app
spec:
  provider:
    previder:
      auth:
        secretRef:
          accessToken:
            name: previder-vault-token
            key: previder-vault-token
```

```
kubectl apply -f hello-app-secretstore.yaml
```

Verify:

```
kubectl -n hello-app get secretstore previder-backend
```

Expected output:

```
NAME               AGE   STATUS   CAPABILITIES   READY
previder-backend   10s   Valid    ReadOnly       True
```

---

# 8. Create an ExternalSecret

`hello-app-externalsecret.yaml`

```
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: hello-app-api
  namespace: hello-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: previder-backend
    kind: SecretStore
  target:
    name: hello-app-api
    creationPolicy: Owner
  data:
  - secretKey: API_KEY
    remoteRef:
      key: hello-app-api-key
```

```
kubectl apply -f hello-app-externalsecret.yaml
```

👉 `remoteRef.key` is the id or description used in step 4. `refreshInterval` controls how often ESO checks the vault for changes — if the secret's value is updated later, the Kubernetes `Secret` is updated automatically within that interval, no `kubectl apply` needed.

---

# 9. Deploy an Example Application

`hello-app-deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: hello-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
      - name: hello-app
        image: paulbouwer/hello-kubernetes:1.10
        envFrom:
        - secretRef:
            name: hello-app-api
```

```
kubectl apply -f hello-app-deployment.yaml
```

---

# 10. Verify

```
kubectl -n hello-app get externalsecret hello-app-api
```

Expected output:

```
NAME            STORE              REFRESH INTERVAL   STATUS         READY
hello-app-api   previder-backend   1h                  SecretSynced   True
```

Confirm the value ended up correctly in the Kubernetes Secret:

```
kubectl -n hello-app get secret hello-app-api -o jsonpath='{.data.API_KEY}' | base64 -d
```

Expected output:

```
s3cr3t-api-key-value
```

---

## ✅ Summary

- Previder Secure Vault is a hosted, multi-tenant secrets service managed through the Previder Portal — nothing needs to be installed inside the cluster for the vault itself.
- An **EnvironmentAdmin** token (from the portal) is only used for setup and to create narrower tokens; it is never placed in the cluster.
- A **ReadOnly** token is what actually goes into the cluster, scoped to reading secrets only — least privilege by design.
- External Secrets Operator's built-in Previder provider authenticates with that token and keeps a Kubernetes `Secret` automatically in sync with what's stored in the vault.
- This pattern (steps 4, 6–10) is the general-purpose reference implementation — repeat it with a different secret and a different application/namespace for any other credential: a database password, an SMTP credential, a webhook token, and so on.

**Note:** `envFrom` only reads a Secret once, when a Pod starts — an updated value in the vault reaches the Kubernetes `Secret` automatically, but running Pods only pick it up after a restart. A tool like [Stakater Reloader](https://github.com/stakater/Reloader) can trigger that restart automatically when the Secret changes.

**Next steps:**
- Create a separate ReadOnly token and `SecretStore` per application/namespace, rather than sharing one token across the whole cluster.
- Manage secrets (`vault-cli secret create/delete`) from a secured workstation or CI pipeline using a ReadWrite token — never from inside the cluster.
