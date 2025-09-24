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

#### 1\. Tell Vault about the Database

First, we enable Vault's database secrets engine and teach it how to connect to our Grimoire as the powerful `admin` user.

```bash
# Start a Vault dev server in a new terminal and set your VAULT_TOKEN, you may have done this already in Part 1.
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

Now, we switch hats to Apprentice Pip's database engineering skills.. The script has been given a short-lived Vault token to perform its tasks.

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

If you ever want to query what encryption keys exist run the following list and read commands in vault.

```bash
vault list encryption_rune/keys
Keys
----
potion-sealing-key

then..

vault read encryption_rune/keys/potion-sealing-key
Key                       Value
---                       -----
allow_plaintext_backup    false
auto_rotate_period        0s
deletion_allowed          false
derived                   false
exportable                false
imported_key              false
keys                      map[1:1758201704]
latest_version            1
min_available_version     0
min_decryption_version    1
min_encryption_version    0
name                      potion-sealing-key
supports_decryption       true
supports_derivation       true
supports_encryption       true
supports_signing          false
type                      aes256-gcm96
```

## The Spell is Complete

And there you have it\! We successfully gave Pip a time limited (ttl) and audited access to a database with the ability to perform powerful cryptography, all without a single long-lived password or encryption key ever being exposed. You've mastered two of the most powerful spells in the HashiCorp Vault grimoire: **Dynamic Secrets** and **Encryption as a Service**.

Of course\! Here is the story of Pip and Greta, formatted as a blog post in Markdown.

-----

# Pip's Guide to Encrypting Large Spellbooks with Vault

*By Pip, the Scripter Sorceress*
*24 September 2025*

One crisp morning in Buckhurst Hill, my friend Greta the Enchanter came to me with a problem. She had just finished digitising her prized possession: a colossal, ancient spellbook. The resulting grimoire file was over a gigabyte, and she needed to store it securely in our shared Vault.

"Pip," she said, looking worried, "I tried to encrypt the grimoire, but Vault threw a fit\! It complained the file was too large. How can I protect my spells?"

She was right. Vault, for its own safety and stability, has a limit on the size of a single secret it can process in one go. But fear not\! I showed her a powerful piece of scripting magic called **"chunking"**.

The idea is simple: instead of trying to enchant the entire massive book at once, we first break it down into hundreds of smaller, manageable pages. We then encrypt each page individually and bundle the encrypted pages together. To read the book again, we simply reverse the process.

Here’s the step-by-step enchantment we used.

-----

## The Encryption Spell: Breaking Down the Grimoire

First, we need to confirm the identity of our grimoire file using a checksum. This ensures that after we decrypt it, we get the exact same file back.

```bash
$ ls -lh pycharm-2025.2.1.1-aarch64.dmg
-rw-r--r--@ 1 user  staff   1.2G 23 Sep 16:38 pycharm-2025.2.1.1-aarch64.dmg

$ md5sum pycharm-2025.2.1.1-aarch64.dmg
ac6e182ed0979f090b1a9a0ad4148c1f  pycharm-2025.2.1.1-aarch64.dmg
```

Now, we can prepare our encryption script.

### The `encrypt.sh` Script

This script takes one argument: the name of the file you want to encrypt.

```bash
#!/bin/bash

# --- Configuration ---
FILE_TO_ENCRYPT=$1
# The path to your transit key in Vault
KEY_PATH="encryption_rune/encrypt/potion-sealing-key"
CIPHERTEXT_FILE="encrypted_bundle.txt"
CHUNK_DIR="temp_chunks"

# 1. Create a temporary directory for chunks
mkdir -p "$CHUNK_DIR"

# 2. Split the large file into 512KB chunks
echo "--> Splitting '$FILE_TO_ENCRYPT' into 512Kb chunks..."
# The -d and -a 4 flags allow for up to 10,000 numeric file chunks (e.g., chunk_0000)
split -b 512k -d -a 4 "$FILE_TO_ENCRYPT" "$CHUNK_DIR/chunk_"

# 3. Encrypt each chunk and append the ciphertext to a single file
echo "--> Encrypting chunks..."
# Clear the output file before starting
> "$CIPHERTEXT_FILE"
for chunk in "$CHUNK_DIR"/chunk_*; do
  # Capture ciphertext and echo it to the file to ensure a newline is added
  ciphertext=$(base64 "$chunk" | vault write -field=ciphertext "$KEY_PATH" plaintext=-)
  echo "$ciphertext" >> "$CIPHERTEXT_FILE"
done

# 4. Clean up the chunks directory
rm -rf "$CHUNK_DIR"

echo "✅ Success! Encrypted data saved to '$CIPHERTEXT_FILE'"
```

> **Note on `base64`:** The `base64` command can have different flags on different systems. The version above works on most Linux distributions. On macOS, you might need `base64 -i "$chunk"`.

With the spell prepared, we run it on Greta’s grimoire:

```bash
chmod +x encrypt.sh
./encrypt.sh pycharm-2025.2.1.1-aarch64.dmg
```

**Output:**

```
--> Splitting 'pycharm-2025.2.1.1-aarch64.dmg' into 512Kb chunks...
--> Encrypting chunks...
✅ Success! Encrypted data saved to 'encrypted_bundle.txt'
```

-----

## The Decryption Counter-Spell: Reassembling the Book

Now that the encrypted pages are bundled in `encrypted_bundle.txt`, Greta needs a way to read her book. This requires a counter-spell to decrypt each page and stitch them back together in the correct order.

### The `decrypt.sh` Script

```bash
#!/bin/bash

# --- Configuration ---
CIPHERTEXT_FILE="encrypted_bundle.txt"
# Note the 'decrypt' path for the key
KEY_PATH="encryption_rune/decrypt/potion-sealing-key"
RESTORED_FILE="decrypted_grimoire.out"

# 1. Read the ciphertext file line by line
echo "--> Decrypting chunks from '$CIPHERTEXT_FILE'..."

# Clear the output file before starting
> "$RESTORED_FILE"
while read -r ciphertext; do
  if [ -n "$ciphertext" ]; then
    echo "$ciphertext" | vault write -field=plaintext "$KEY_PATH" ciphertext=- | base64 --decode >> "$RESTORED_FILE"
  fi
done < "$CIPHERTEXT_FILE"

echo "✅ Success! Restored file saved to '$RESTORED_FILE'"
```

> **Note on `base64`:** The flag for decoding is typically `--decode` or `-d`. On macOS, it's `-D`.

We execute the decryption script:

```bash
chmod +x decrypt.sh
./decrypt.sh
```

**Output:**

```
--> Decrypting chunks from 'encrypted_bundle.txt'...
✅ Success! Restored file saved to 'decrypted_grimoire.out'
```

-----

## Verifying the Magic

The final, crucial step is to ensure the reassembled grimoire is a perfect copy of the original. We use the same checksum spell as before.

```bash
$ md5sum decrypted_grimoire.out
ac6e182ed0979f090b1a9a0ad4148c1f  decrypted_grimoire.out
```

The checksums match\! Greta’s massive spellbook was successfully protected and restored using the power of Vault and a little bit of scripting magic. This chunking method can be used to protect any large file, no matter its size.

Happy enchanting\!

\-Pip
## Next..

> **[Part 3: PKI and Mutual TLS (mTLS)](./part3.md)**
    - Set up Vault as a Certificate Authority (CA) to issue TLS certificates for your PostgreSQL server and clients, enabling a secure, passwordless, and verifiable mTLS connection.
