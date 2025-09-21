# The Alchemist's Vault: A Magical Guide to Dynamic Secrets and Encryption - Part 2

Welcome, aspiring security wizards! Today, we'll journey to the Grand Academy of Arcane Arts to solve a modern magical problem. We'll learn how to use the powerful magic of **HashiCorp Vault** to grant secure, temporary access to a secret database and perform powerful encryption without ever revealing the master keys.

### Our Magical Quest

Our story follows **Apprentice Pip**, a junior wizard tasked with brewing a powerful "Elixir of Invisibility." The main recipe is stored in the school's magical database, "The Grand Grimoire." Pip has made a breakthrough, discovering a new, secret ingredient that will make the potion ten times more powerful!

The challenge? Pip's automated brewing script needs to:
1.  Securely access The Grand Grimoire to fetch the base recipe.
2.  Encrypt the new secret ingredient before saving the completed formula.

The Headmaster won't just hand over the master keys to the Grimoire. We need a better, more magical way.

### The Cast of Characters

* 🧙‍♂️ **Headmaster Alistair (You, the Admin):** The all-powerful wizard who controls the school's secrets. You'll configure The Alchemist's Vault.
* 🤖 **Apprentice Pip (Your Application/Script):** A script that needs temporary access to the database and encryption capabilities.
* 📚 **The Grand Grimoire (PostgreSQL Database):** Contains all the known potion recipes.
* 🔐 **The Alchemist's Vault (Vault Server):** Our central source of all magical secrets.
* ✨ **The Encryption Rune (Vault's Transit Engine):** A service that can encrypt and decrypt data without ever revealing its own secret key.

---

## Act I: Headmaster Alistair Configures the Vault 🧙‍♂️

As the Headmaster, your first job is to prepare the academy's infrastructure.

### Preparing the Grand Grimoire (PostgreSQL)

First, we need our database. We'll use Docker Compose to quickly conjure a PostgreSQL instance and a pgAdmin interface.

Save the following as `compose.yml`:

```yaml
services:
  postgres:
    container_name: container-pg
    image: postgres
    hostname: localhost
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: root
      POSTGRES_DB: test_db
    volumes:
      - postgres-data:/var/lib/postgresql/data
    restart: unless-stopped

  pgadmin:
    container_name: container-pgadmin
    image: dpage/pgadmin4
    depends_on:
      - postgres
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: root
    restart: unless-stopped

volumes:
  postgres-data:
````

Now, start the services:

```bash
docker-compose up -d
```

With our database running, let's prepare it for our potions. We'll connect as the `admin` user, create the `grand_grimoire` database, and set up a secure "readers" group. This is a best practice that ensures we only grant the permissions needed.

```bash
# Connect as the admin user to the default postgres database
# The password is 'root'
psql -h localhost -U admin -d postgres
```

Inside `psql`, run the following SQL commands:

```sql
create database grand_grimoire;
\c grand_grimoire
create table recipes (id SERIAL PRIMARY KEY, name VARCHAR(50) NOT NULL, ingredients VARCHAR(100) NOT NULL);

-- Create a group role to hold permissions
CREATE ROLE recipe_readers;

-- Grant this group the ability to connect and use the schema
GRANT CONNECT ON DATABASE grand_grimoire TO recipe_readers;
GRANT USAGE ON SCHEMA public TO recipe_readers;

-- Grant the group SELECT on all CURRENT and FUTURE tables
GRANT SELECT ON ALL TABLES IN SCHEMA public TO recipe_readers;
ALTER DEFAULT PRIVILEGES FOR ROLE admin IN SCHEMA public GRANT SELECT ON TABLES TO recipe_readers;

-- **The final fix:** Ensure Row-Level Security is not blocking access
ALTER TABLE recipes DISABLE ROW LEVEL SECURITY;

-- Add our base recipe to the Grimoire
INSERT INTO recipes VALUES (1, 'Elixir of Invisibility', 'Moon dust, shadow essence');
```

### Preparing the Alchemist's Vault

With the Grimoire ready, Headmaster Alistair can now configure the Vault.

#### 1\. Teach Vault about the Database

First, we enable Vault's database secrets engine and teach it how to connect to our Grimoire as the powerful `admin` user.

```bash
# Start a Vault dev server in a new terminal and set your VAULT_TOKEN
vault server -dev

# Enable the database secrets engine
vault secrets enable database

# Configure the connection to our PostgreSQL Grimoire
vault write database/config/grand_grimoire \
    plugin_name=postgresql-database-plugin \
    allowed_roles="apprentice-role" \
    connection_url="postgresql://admin:root@localhost:5432/postgres?sslmode=disable"
```

#### 2\. Create a Role for Apprentices

Next, we define a role for apprentices. This role tells Vault how to create temporary users. Our apprentices will only get read-only access for two minutes.

```bash
# Create a role for apprentices with specific permissions
vault write database/roles/apprentice-role \
    db_name="grand_grimoire" \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT recipe_readers TO \"{{name}}\";" \
    default_ttl="2m" \
    max_ttl="5m" \
    username_template="v-{{.RoleName | truncate 8}}-{{random 8}}"
```

#### 3\. Activate the Encryption Rune

Finally, we enable the `transit` secrets engine. This creates a powerful encryption service that applications can use without ever seeing the underlying key.

```bash
# Enable the transit secrets engine at the path "encryption_rune"
vault secrets enable -path=encryption_rune transit

# Create a named encryption key for sealing potion formulas
vault write -f encryption_rune/keys/potion-sealing-key
```

"Excellent. The enchantments are set. The Vault is ready for Apprentice Pip."

-----

## Act II: Apprentice Pip's Quest 🤖

Now, we switch hats to Apprentice Pip's automated script. The script has been given a short-lived Vault token to perform its tasks.

### 1\. Request Temporary Database Credentials

First, the script needs the recipe from the Grimoire. It asks the Vault for a temporary credential.

```bash
# Pip's script asks for credentials based on the apprentice role
vault read database/creds/apprentice-role
```

Vault returns a unique, dynamically generated username and password\!

```
Key               Value
---               -----
lease_id          database/creds/apprentice-role/01Et2G3il8uEMUSMdUxEuK6J
lease_duration    1m
lease_renewable   true
password          osVO6exdESF-v2HEgMCR
username          v-token-apprenti-RdAoBOupDtO4D3Be8p4m-1758201316
```

Pip's script can now use these credentials to connect and fetch the recipe.

```bash
# Use the dynamic username and password from the output above
psql -h localhost -d grand_grimoire -c "select * from recipes;" -U v-token-apprenti-RdAoBOupDtO4D3Be8p4m-1758201316
```

### 2\. Encrypt the New Secret Ingredient

Now for Pip's great discovery\! The new ingredient is 'Powdered Dragon's Tear'. We must encrypt this before saving it. The script sends the secret to the Encryption Rune.

```bash
# Pip's script asks the transit engine to encrypt the secret
# (The secret must be base64 encoded to be sent over the API)
SECRET_INGREDIENT=$(echo -n "Powdered Dragon's Tear" | base64)

vault write encryption_rune/encrypt/potion-sealing-key plaintext=$SECRET_INGREDIENT
```

The rune returns the powerful ciphertext:

```
Key            Value
---            -----
ciphertext     vault:v1:u0QDo+lXr3oWwHYCu23G1ixZkUgoJxw7s13pywnLwt/MU9ot0pTDPbe2uJfxrF6hYp0=
key_version    1
```

### 3\. Save and Reveal the Secret

Pip's script can now safely save this encrypted value back into the Grimoire.

```bash
# Connect as the admin to insert the new recipe
psql -h localhost -U admin -d grand_grimoire
```

Inside `psql`:

```sql
insert into recipes values(2, 'Pips Magic Potion', 'vault:v1:u0QDo+lXr3oWwHYCu23G1ixZkUgoJxw7s13pywnLwt/MU9ot0pTDPbe2uJfxrF6hYp0=');
```

Later, an authorized user can retrieve the encrypted value and ask the rune to reveal its secret.

```bash
vault write encryption_rune/decrypt/potion-sealing-key ciphertext="vault:v1:u0QDo+lXr3oWwHYCu23G1ixZkUgoJxw7s13pywnLwt/MU9ot0pTDPbe2uJfxrF6hYp0=" -format=json | jq -r .data.plaintext | base64 -d | strings
```

Output:

```
Powdered Dragon's Tear
```

-----

## The Spell is Complete

And there you have it\! We successfully gave an application temporary, audited access to a database and the ability to perform powerful cryptography, all without a single long-lived password or encryption key ever being exposed. You've mastered two of the most powerful spells in the HashiCorp Vault grimoire: **Dynamic Secrets** and **Encryption as a Service**.

> **[Part 1: Dynamic Database Secrets](./part1.md)**
    - Learn how to configure Vault to dynamically generate temporary, on-demand credentials for a PostgreSQL database, eliminating the need for long-lived static passwords.

> **[Part 3: PKI and Mutual TLS (mTLS)](./part3.md)**
    - Set up Vault as a Certificate Authority (CA) to issue TLS certificates for your PostgreSQL server and clients, enabling a secure, passwordless, and verifiable mTLS connection.
