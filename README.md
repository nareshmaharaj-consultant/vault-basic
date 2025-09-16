# Getting Started with HashiCorp Vault: A Practical Guide

## Overview

This guide provides a hands-on introduction to HashiCorp Vault, covering basic operations, authentication, policies, and secrets management. It's based on real-world usage patterns and common pain points.

## 1. Starting a Vault Server

### Development Server (Simple)
```bash
vault server -dev

# Set environment variables
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_DEV_ROOT_TOKEN_ID='hvs.U1k8q1L9yl6A0hG0KtDquoyO'
export VAULT_DEV_ROOT_TOKEN='hvs.U1k8q1L9yl6A0hG0KtDquoyO'

# Check status
vault status
```

### Development Server with TLS
```bash
vault server -dev -dev-root-token-id root -dev-tls

export VAULT_ADDR='https://127.0.0.1:8200'
export VAULT_CACERT='/path/to/vault-ca.pem'
```

## 2. Authentication and Tokens

### Login with Root Token
```bash
vault login root
```

### Understanding Vault Tokens
Tokens are credentials issued after successful authentication. They determine what operations a client can perform.

#### Create Different Token Types
```bash
# Basic token with TTL
vault token create -policy=default -ttl=60s

# Orphan token (not tied to parent token)
vault token create -policy=default -ttl=60s -orphan

# Periodic token (can be renewed within max TTL)
vault token create -policy="default" -period=1m -explicit-max-ttl=5m

# Renew a token
vault token renew -accessor <accessor_id>
```

### Token Lookup
```bash
vault token lookup -accessor <accessor_id>
```

## 3. Policies

Policies define what operations a token can perform. They are attached to tokens after authentication.

### Basic Policy Example
```bash
vault policy write my-policy - <<EOF
path "secret/data/my-app/config" {
  capabilities = ["read"]
}
EOF
```

### KV v2 Policy Nuance
**Important**: KV v2 engines require permissions on both `data` and `metadata` paths:

```bash
vault policy write my-policy - <<EOF
# For the actual secret data
path "/my-secrets/data/names" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# For metadata operations (listing, reading metadata)
path "/my-secrets/metadata/names" {
  capabilities = ["list", "read"]
}

# To list all secrets in the engine
path "/my-secrets/metadata/" {
  capabilities = ["list"]
}
EOF
```

### Policy Patterns
```hcl
# Wildcard matching
path "dev-secrets/+/creds" {
  capabilities = ["create", "list", "read", "update"]
}

# Template with entity name
path "dev-secrets/+/creds/{{identity.entity.name}}" {
  capabilities = ["create", "list", "read", "update"]
}

# Required parameters constraint
path "dev-secrets/+/creds" {
  capabilities = ["create", "list", "read", "update"]
  required_parameters = ["username"]
}
```

### Attach Policy to Token
```bash
vault token create -policy="my-policy" -ttl=10m
```

## 4. Authentication Methods

### Enable Userpass Auth
```bash
vault auth enable userpass
```

### Create User
```bash
vault write auth/userpass/users/naresh \
  password="super-secret-password" \
  policies="my-policy"
```

### List Users
```bash
vault list auth/userpass/users
```

### User Login
```bash
vault login -method=userpass username=naresh
```

### Logout
```bash
vault token revoke -self
```

## 5. Secrets Management

### List Enabled Secrets Engines
```bash
vault secrets list
```

### Enable KV v2 Secrets Engine
```bash
vault secrets enable -path=/my-secrets -version=2 kv
```

### Basic Secret Operations

#### Write/Update Secrets
```bash
vault kv put my-secrets/names A=B
vault kv put my-secrets/users name=naresh
```

#### Read Secrets
```bash
vault kv get -mount=my-secrets names
vault read my-secrets/data/names
```

#### List Secrets
```bash
vault kv list -mount=my-secrets
vault list my-secrets/metadata
```

## 6. Common Issues and Solutions

### Permission Denied Errors
If you get `403 permission denied` when trying to list secrets, remember:
- Use `vault kv get` to read a specific secret's contents
- Use `vault kv list` to see what secrets exist
- Ensure your policy grants permissions on both `data` and `metadata` paths

### KV v2 Path Structure
Remember the KV v2 structure:
- `mount-point/data/secret-name` - for the actual secret values
- `mount-point/metadata/secret-name` - for metadata about the secret
- `mount-point/metadata/` - for listing available secrets

## 7. Best Practices

1. **Use appropriate token types**: Service tokens for machines, periodic tokens for long-running services
2. **Follow principle of least privilege**: Grant only necessary capabilities
3. **Retain old KMS key versions**: Needed for restoring old backups
4. **Test policies thoroughly**: Permission issues are common with KV v2
5. **Use environment variables**: Set `VAULT_ADDR` and authentication tokens appropriately

## Conclusion

This guide covers the essential Vault operations you'll need for development and testing. Remember that production setups require proper sealing/unsealing, high availability configuration, and secure storage backends.

The key to mastering Vault is understanding the relationship between authentication methods, tokens, policies, and secret engines - particularly the nuances of the KV v2 engine's path structure.
