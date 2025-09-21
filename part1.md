# HashiCorp Vault: A Practical Guide from First Steps to "Aha!" Moments - Part 1

This guide walks through a real hands-on session with HashiCorp Vault, from starting a server to wrestling with and finally understanding the nuances of policies and secrets in the KV v2 engine. It includes all the commands, outputs, and hard-learned lessons.

## 1. Starting a Vault Dev Server

The easiest way to start experimenting is with a development server. **Warning: This is for learning only and is not secure for production.**

### Option 1: Standard Dev Server
This generates a root token and unseal key for you.
```bash
vault server -dev
```
The server starts and outputs crucial credentials. **You must export these environment variables.**
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_DEV_ROOT_TOKEN_ID='hvs.U1k8q1L9yl6A0hG0KtDquoyO'
export VAULT_TOKEN='hvs.U1k8q1L9yl6A0hG0KtDquoyO' # Usually the same as DEV_ROOT_TOKEN_ID
```
Check the server status:
```bash
vault status
```
Expected output:
```
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.20.3
Build Date      2025-08-27T10:53:27Z
Storage Type    inmem
Cluster Name    vault-cluster-a5ac0805
Cluster ID      5c8d23ce-401c-b165-a003-d434dc6e90d7
HA Enabled      false
```

### Option 2: Dev Server with Custom Root Token & TLS
Better for simulating a more realistic environment.
```bash
vault server -dev -dev-root-token-id root -dev-tls
```
Export the required variables, noting the HTTPS address and CA certificate.
```bash
export VAULT_ADDR='https://127.0.0.1:8200'
export VAULT_CACERT='/var/folders/pr/d8jpl0q14d375nfb6nk135d40000gp/T/vault-tls1652600587/vault-ca.pem'
```

## 2. Authentication and Token Basics

### Logging In
Before any operation, you must authenticate. As the root user:
```bash
vault login root
```
Output:
```
Key                  Value
---                  -----
token                root
token_accessor       d2UhKlFeQfKvwFPRqNkWSfML
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

### What is a Vault Token?
A token is a credential issued upon successful authentication. $\color{Blue}{\textbf{Every operation in Vault requires a token.}}$ The `token_accessor` is a reference that can be used to manage the token (renew, revoke, lookup) without knowing the token itself.

**Look up a token using its accessor:**
```bash
vault token lookup -accessor d2UhKlFeQfKvwFPRqNkWSfML
```

### Creating Tokens with Different Properties
Root is powerful, so you would be advised to create limited tokens for specific use cases.

**1. Basic Token with Short TTL:**
```bash
vault token create -policy=default -ttl=60s
```
**2. Orphan Token:**
Not tied to its parent token. Useful for long-running services or emergency scenarios to prevent accidental revocation.
```bash
vault token create -policy=default -ttl=60s -orphan
```
**3. Periodic Token:**
Can be renewed repeatedly within its explicit max TTL. Ideal for long-running services.
```bash
vault token create -policy="default" -period=1m -explicit-max-ttl=5m
# Key: This token can be renewed every 5 mins but last for only 1m
```
**Renewing a Token:**
```bash
vault token renew -accessor qI4rkcYPWgZSDDeHjA1iA8bE
```

## 3. Policies: Defining Permissions

Policies are attached to tokens and define exactly what paths and operations (capabilities) are permitted.

### Writing a Basic Policy
Policies are written in HCL (HashiCorp Configuration Language).
```bash
vault policy write my-policy - <<EOF
# Allow read access to a specific secret path
path "secret/data/my-app/config" {
  capabilities = ["read"]
}
EOF
```

### The KV v2 Policy "Head-Banger"
A critical lesson learned: The KV v2 engine has a different data model. Writing a policy for `data` is not enough. You need permissions on `metadata` to perform listing operations.

**The Initial (Faulty) Policy:**
This policy seems logical but will cause permission errors.
```bash
vault policy write my-policy - <<EOF
# To manage the secret named 'names'
path "/my-secrets/data/names" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOF
```

### Advanced Policy Patterns
**Wildcards:** Use `+` to match any single directory segment.
```hcl
path "dev-secrets/+/creds" {
  capabilities = ["create", "list", "read", "update"]
}
```
**Templating:** Use identity information in paths.
```hcl
path "dev-secrets/+/creds/{{identity.entity.name}}" {
  capabilities = ["create", "list", "read", "update"]
}
```
**Parameter Constraints:** Force specific data keys to be present.
```hcl
path "dev-secrets/+/creds" {
  capabilities = ["create", "list", "read", "update"]
  required_parameters = ["username"] # Secret must have a 'username' key
}
```

## 4. Configuring Authentication: Userpass

Enable the `userpass` authentication method to allow username/password logins.
```bash
vault auth enable userpass
```

Create a user and attach the policy we created:
```bash
vault write auth/userpass/users/naresh \
  password="super-secret-password" \
  policies="my-policy"
```

List the users:
```bash
vault list auth/userpass/users
# Keys
# -----
# naresh
```

Login as the new user:
```bash
vault login -method=userpass username=naresh
Password (will be hidden):
```
Success output shows the token issued to `naresh`, which has the `my-policy` and `default` policies attached.

## 5. Secrets Engines: Using KV v2

Vault supports many secrets engines (databases, PKI, etc.). The Key/Value store is the most common starting point.

### Enable the KV v2 Secrets Engine
The `-version=2` flag is crucial for enabling the newer engine with versioning and metadata.
```bash
vault secrets enable -path=/my-secrets -version=2 kv
```

### Basic Secret Operations
**Write a Secret:**
```bash
vault kv put my-secrets/names A=B
```
**Output:**
```
==== Secret Path ====
my-secrets/data/names

======= Metadata =======
Key                Value
---                -----
created_time       2025-09-15T15:28:36.975106Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

**Read a Secret using the `kv` command:**
```bash
vault kv get -mount=my-secrets names
```

**Read a Secret using the API path:**
```bash
vault read my-secrets/data/names
```
**Output:**
```
Key         Value
---         -----
data        map[A:B]
metadata    map[created_time:2025-09-15T15:28:36.975106Z custom_metadata:<nil> deletion_time: destroyed:false version:1]
```

**Write Another Secret:**
```bash
vault kv put my-secrets/users name=naresh
```

### The Permission Problem: A Case Study

This is where the real learning happened. After logging in as `naresh`, trying to list secrets failed.

**The Error:**
```bash
vault kv list -mount=my-secrets
# Error listing my-secrets/metadata/names: Error making API request.
# URL: GET http://127.0.0.1:8200/v1/my-secrets/metadata/names?list=true
# Code: 403. Errors:
# * 1 error occurred:
#  * permission denied
```

**The Reason:**
The command `vault kv list` makes a `LIST` API call to the path `my-secrets/metadata/`. Our initial policy only granted permissions on `my-secrets/data/names`, not on the required `metadata` path.

**The Solution:**
The policy must be updated to grant `list` capability on the metadata path.
```bash
# As the root user, update the policy
vault policy write my-policy - <<EOF
# Read/write the secret data itself
path "/my-secrets/data/names" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
# List the secret and read its metadata
path "/my-secrets/metadata/names" {
  capabilities = ["list", "read"]
}
# Optional: List all secrets under 'my-secrets'
path "/my-secrets/metadata/" {
  capabilities = ["list"]
}
EOF
```

**Testing the Fix:**
1.  The user `naresh` must log out and back in to get the updated policy.
    ```bash
    vault token revoke -self # Run as naresh
    vault login -method=userpass username=naresh # Log back in
    ```
2.  Now, listing works:
    ```bash
    vault kv list -mount=my-secrets
    # Keys
    # ----
    # names
    # users
    ```
3.  But trying to read the `users` secret still failed because the policy was too specific.
    ```bash
    vault kv get -mount=my-secrets users
    # Error reading my-secrets/data/users: Error making API request.
    # Code: 403. Errors:
    # * 1 error occurred:
    #  * permission denied
    ```

**The Final Policy Fix:**
The policy had to be updated again to include the new `users` path, demonstrating the iterative process of policy management.
```bash
vault policy write my-policy - <<EOF
# Read/write the secret data itself
path "/my-secrets/data/names" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "/my-secrets/data/users" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
# List the secret and read its metadata
path "/my-secrets/metadata/names" {
  capabilities = ["list", "read"]
}
path "/my-secrets/metadata/users" {
  capabilities = ["list", "read"]
}
# List all secrets under 'my-secrets'
path "/my-secrets/metadata/" {
  capabilities = ["list"]
}
EOF
```

After this final update and a new login, the user `naresh` could successfully list and read all the intended secrets.

## Key Takeaways and Best Practices

1.  **KV v2 Paths are King:** Remember the structure. You nearly always need permissions on both `data/<secret>` and `metadata/<secret>` paths.
2.  **Principle of Least Privilege:** Start with restrictive policies and expand them as needed, exactly as we did iteratively.
3.  **Policies are Bound to Tokens:** When you update a policy, existing tokens are *not* updated. Users must log out and back in to receive a new token with the updated policy.
4.  **Use `vault kv` Commands:** The `vault kv` command family (e.g., `vault kv get`, `vault kv list`) is more user-friendly than trying to remember the underlying API paths (`my-secrets/data/...`).
5.  **Expect Permission Issues:** A `403` error almost always means your policy is missing a capability on a specific path. Use the error message to see what path was being accessed and adjust your policy accordingly.

This journey from a basic server setup through debugging policy errors encapsulates the core experience of getting started with Vault. The key is understanding the relationship between auth methods, tokens, policies, and the specific API structure of the secrets engines you use.

> **[Part 2: Encryption as a Service](./part2.md)**
     - Discover how to use Vault's Transit Secrets Engine to encrypt and decrypt sensitive data without ever exposing the encryption keys to your application.
