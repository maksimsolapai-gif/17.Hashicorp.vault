# Homework Assignment Report: Hashicorp Vault in Docker and Kubernetes

## 📁 Repository Structure
```text
17.Hashicorp.vault/
├── docker-compose.yml       # Launching Vault in dev mode (Assignment 1)
├── app-read-policy.hcl      # HCL access policy for AppRole (Assignment 1)
├── vault-values.yaml        # Helm chart configuration for Vault with Raft (Assignment 2)
├── vso-resources.yaml       # VSO manifests (Connection, Auth, StaticSecret)
└── app-deployment.yaml      # Test application consuming the secrets
```

---

## 🔐 Homework Assignment 1: Vault in Docker Compose — KV, policy, AppRole

### Vault Status in Dev-mode
```
docker compose exec -it vault sh
max@ubuntu:~/Project/17.Vault$ docker compose exec -it vault sh

/ # export VAULT_ADDR='http://127.0.0.1:8200'
/ # export VAULT_TOKEN='root'
/ #
/ # vault status         # Checking status

Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.17.6
Build Date      2024-09-24T19:48:40Z
Storage Type    inmem
Cluster Name    vault-cluster-f833ba44
Cluster ID      929cf367-41e5-8399-9743-65ba0bb9f948
HA Enabled      false
```

### Description of the `vault status` output:
•	Sealed: false. Since it was launched in dev-mode, Vault automatically initializes and unseals upon startup.

•	Storage: inmem (in-memory). Data is not preserved when the container restarts.


## Working with KV v2 secrets
vault kv put secret/app/db username=alice password=s3cret
<img width="975" height="319" alt="image" src="https://github.com/user-attachments/assets/e0697dcf-c9e6-4048-8ce9-01be837c920f" />

```
vault kv get secret/app/db
vault kv get -field=password secret/app/db
vault kv put secret/app/db username=admin password=new_awesome_pass
vault kv get -version=1 secret/app/db 
vault kv metadata get secret/app/db
```
<img width="975" height="310" alt="image" src="https://github.com/user-attachments/assets/43f4d09d-0a02-41ba-ad40-a22460404eff" />

### The difference between delete, undelete, and destroy

```
/ # vault kv delete secret/app/db
Success! Data deleted (if it existed) at: secret/data/app/db
```
<img width="975" height="384" alt="image" src="https://github.com/user-attachments/assets/d468181c-ee87-4311-8732-202df6406338" />

###  Difference between `delete`, `undelete` and `destroy`
* **`vault kv delete`** — Performs a *Soft Delete* of a specific secret version. The data is marked as deleted but physically remains in the database. The version can be restored.
* **`vault kv undelete`** — Restores a previously soft-deleted version of a secret, making it accessible for reading again.
* **`vault kv destroy`** — Permanently and irreversibly removes the data of the specified version from the disk/storage. It is impossible to recover this version.

###  Theoretical Questions (Assignment 1)
1. **Why dev mode must never be used in production:**
   * Data is stored in-memory (`inmem`) and is lost when the container restarts.
   * Vault is automatically unsealed and unseal keys are not generated, which violates Shamir's Secret Sharing security model.
   * TLS is disabled by default, meaning traffic is transmitted over unencrypted HTTP.
2. **Why an application should authenticate with AppRole instead of the root token:**
   * **Principle of Least Privilege:** Roles are granted specific capabilities restricted to certain paths. A root token has absolute privileges (can delete the cluster or read unauthorized secrets).
   * **Audit and Revocation:** AppRole tokens are short-lived (short TTL) and can be easily revoked individually without re-keying the core system.

### Security Policies (HCL)
```
app-read-policy.hcl:

cat > /tmp/app-read-policy.hcl <<'EOF'
path "secret/data/app/*" {
  capabilities = ["read"]
}
path "secret/metadata/app/*" {
  capabilities = ["read", "list"]
}
EOF

vault policy write app-read /tmp/app-read-policy.hcl
vault policy read app-read

```
<img width="975" height="325" alt="image" src="https://github.com/user-attachments/assets/5cfa9640-eefd-45df-b013-22f904c63404" />



---

##  Homework Assignment 2: Vault in Kubernetes and delivering a secret with VSO

### Pod Status at Startup
The Vault pod transitions to the `1/1` Ready state **only after being unsealed** (`vault operator unseal`). 
* **Why Vault starts sealed:** This is a fundamental security design of Vault. All data on disk is encrypted using a Master Key. Upon startup, Vault deliberately isolates itself from the key material in memory to ensure that if the server or disks are compromised, the attacker cannot read the data. Vault can only be unsealed by a trusted group of administrators providing a quorum of keys.

### ❓ Theoretical Questions (Assignment 2)

1. **What happens to the environment variables of a Pod that is already running after secret rotation:**
   * **Result:** Environment variables inside a running container (`printenv`) **do not change**, even though the native Kubernetes Secret is successfully updated on the fly by the VSO operator.
   * **Why:** Environment variables for a Linux process are initialized strictly at container startup. Changes to the underlying `Secret` object are not dynamically pushed into already running processes by the OS.
   * **What to do:** Enable VSO's built-in automated rollout restart feature upon secret updates (`rollout restart`), or mount the secret as a file (`volumeMounts`), since Kubernetes natively supports live updates for mounted file volumes.

2. **How Vault Secrets Operator (VSO) compares to Sealed Secrets (Bitnami):**
   * **VSO Pros:** Supports automated rotation and dynamic secrets (e.g., temporary database credentials), provides robust audit logging (who requested what and when), and eliminates the need to commit encrypted secrets into Git.
   * **VSO Cons:** High coupling (Tight Coupling) — if the Vault cluster goes down, new pods cannot start up because VSO won't be able to generate the native Kubernetes Secret.

3. **What happens if you lose 3 of the 5 unseal keys but keep a full backup of the storage:**
   * The cluster **cannot be unsealed**, and the data will remain permanently locked. Shamir's scheme strictly requires a threshold quorum of keys (in this case, 3 out of 5). Since 3 are lost, only 2 remain, which is insufficient. The storage backup contains encrypted data that is useless without the Master Key. The only recovery path is a complete re-initialization of Vault, resulting in data loss.
