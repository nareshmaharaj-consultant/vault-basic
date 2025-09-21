# Securing the Grand Grimoire: A Guide to Vault PKI and PostgreSQL mTLS - Part 3

This document details the process of securing a PostgreSQL database using Mutual TLS (mTLS), where both the server and the clients must present valid, trusted certificates to establish a connection. We will use HashiCorp Vault as our Certificate Authority (CA) to dynamically issue all necessary certificates.

## The Story 🧙‍♂️

Headmaster Alistair of the Grand Academy of Arcane Arts has issued a new edict: all connections to the magical database, "The Grand Grimoire," must be secured with certificates from the Alchemist's Vault. This guide follows the steps to set up this magical security, first for the Headmaster himself, and then for a new apprentice, `appuser`.

### Prerequisites

Before you begin, ensure you have the following tools installed:
* Docker and Docker Compose
* HashiCorp Vault CLI
* PostgreSQL Client (`psql`)
* `jq` (for parsing JSON)

---

## Part 1: Setting Up the Environment

### 1.1. Prepare the PostgreSQL Database

If you haven't already created the postgresql database in part 2, here it is again. 
Create our PostgreSQL instance using Docker Compose.

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

Start the services:

```bash
docker-compose up -d
```

### 1.2. Create and Configure the "Grand Grimoire" Database

We already did this is previous section (part2) but here it is again.
Connect to the running instance and set up our database, table, and a dedicated role for read-only access.

```bash
# Connect as the 'admin' superuser (password: root)
psql -h localhost -U admin -d postgres
```

Inside the `psql` prompt, run the following SQL:

```sql
-- Create our database
create database grand_grimoire;
\c grand_grimoire

-- Create our table for recipes
create table recipes (id SERIAL PRIMARY KEY, name VARCHAR(50) NOT NULL, ingredients VARCHAR(100) NOT NULL);

-- Create a group role that will hold the read-only permissions
CREATE ROLE recipe_readers;

-- Grant this group the necessary permissions to connect and read
GRANT CONNECT ON DATABASE grand_grimoire TO recipe_readers;
GRANT USAGE ON SCHEMA public TO recipe_readers;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO recipe_readers;
ALTER DEFAULT PRIVILEGES FOR ROLE admin IN SCHEMA public GRANT SELECT ON TABLES TO recipe_readers;

-- Disable Row-Level Security to rely on GRANT permissions
ALTER TABLE recipes DISABLE ROW LEVEL SECURITY;

-- Insert some initial data
INSERT INTO recipes VALUES (1, 'Elixir of Invisibility', 'Moon dust, shadow essence');
```

-----

## Part 2: Configuring Vault as a Certificate Authority

With the database ready, we'll now configure Vault to act as the CA for our academy.

### 2.1. Start Vault and Enable the PKI Engine

```bash
# Start a Vault dev server in a new terminal
vault server -dev

# Set your VAULT_ADDR and VAULT_TOKEN environment variables from the output above
export VAULT_ADDR='https://127.0.0.1:8200'
export VAULT_TOKEN='...'

# Enable the PKI secrets engine
vault secrets enable pki

# Tune the root CA's max lease to 10 years
vault secrets tune -max-lease-ttl=87600h pki

# Generate the root certificate, saving it to a file just for reference only. We wont actually use AcademyCA.crt.
vault write -field=certificate pki/root/generate/internal \
    common_name="Grand Academy CA" \
    ttl=87600h > AcademyCA.crt
```

### 2.2. Configure Certificate URLs and Roles

Define the URLs for certificate distribution and create roles that dictate the rules for issuing certificates.

```bash
# Configure the URLs for the CRL and issuing certificates
vault write pki/config/urls \
    issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
    crl_distribution_points="$VAULT_ADDR/v1/pki/crl"

# Create a flexible role for our various users and services
vault write pki/roles/apprentice-role \
    allowed_domains="academy.local,admin,appuser" \
    allow_subdomains=true \
    allow_bare_domains=true \
    max_ttl="1h"

# Create a dedicated, more privileged role for the admin user
vault write pki/roles/admin-role \
    allowed_domains="admin" \
    allow_bare_domains=true \
    max_ttl="24h"
```

-----

## Part 3: Configuring PostgreSQL for mTLS

Now we'll enchant the Grimoire to require valid certificates for all connections.

### 3.1. Generate and Deploy the Server Certificate

The PostgreSQL server itself needs a certificate to prove its identity. We issue one from Vault that is valid for both its internal and external names (`localhost`).

```bash
# Request a server certificate from Vault with Subject Alternative Names (SANs)
vault write -format=json pki/issue/apprentice-role \
    common_name="grimoire.academy.local" \
    alt_names="localhost" \
    ip_sans="127.0.0.1" > postgres-cert.json

# Extract the certificate, key, and CA into a directory
mkdir -p server-certs
jq -r .data.private_key < postgres-cert.json > server-certs/server.key
jq -r .data.certificate < postgres-cert.json > server-certs/server.crt
jq -r .data.issuing_ca < postgres-cert.json > server-certs/ca.crt
chmod 600 server-certs/server.key

# Copy the server certificates into the Docker container
ls -1 server-certs | while read -r line; do docker cp server-certs/$line container-pg:/var/lib/postgresql/data/; done

# Set the correct permissions on the key inside the container
docker exec -it container-pg chmod 600 /var/lib/postgresql/data/server.key
```

### 3.2. Update PostgreSQL Configuration

We need to edit `postgresql.conf` and `pg_hba.conf` to enable and enforce SSL.

**In `postgresql.conf`, set:**

```ini
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'ca.crt'
```

**In `pg_hba.conf`, add these lines at the top:**

```ini
# TYPE    DATABASE   USER      ADDRESS        METHOD       OPTIONS
hostssl   all        all       0.0.0.0/0      cert         clientcert=verify-full
hostnossl all        all       0.0.0.0/0      reject
local     all        all                      peer
```

To apply these changes, copy the modified files into the container and restart it.

```bash
# Example of copying the files back (assuming they are in the current directory)
docker cp postgresql.conf container-pg:/var/lib/postgresql/data/postgresql.conf
docker cp pg_hba.conf container-pg:/var/lib/postgresql/data/pg_hba.conf

# Ensure correct ownership inside the container
docker exec -it container-pg chown -R postgres:postgres /var/lib/postgresql/data/

# Restart PostgreSQL to load the new config and certificates
docker restart container-pg
```

### 3.3. Verify mTLS is Enforced

A connection attempt without a valid client certificate should now fail. This error is a sign of success\!

```bash
psql -U admin -h localhost -d postgres -c '\l'
```

```
psql: error: connection to server at "localhost" (::1), port 5432 failed: FATAL:  connection requires a valid client certificate
```

-----

## Part 4: Connecting with Client Certificates

### 4.1. Connecting as the `admin` User

Let's test our setup by connecting as the superuser `admin`.

```bash
# Issue the client certificate for the admin user
vault write -format=json pki/issue/admin-role \
    common_name="admin" > client-admin-cert.json

# Extract the files
mkdir -p client-certs
jq -r .data.private_key < client-admin-cert.json > client-certs/admin-client.key
jq -r .data.certificate < client-admin-cert.json > client-certs/admin-client.crt
jq -r .data.issuing_ca < client-admin-cert.json > client-certs/ca.crt
chmod 600 client-certs/admin-client.key

# Connect using the certificate
psql "host=localhost dbname=grand_grimoire user=admin \
sslmode=verify-full \
sslcert=$(pwd)/client-certs/admin-client.crt \
sslkey=$(pwd)/client-certs/admin-client.key \
sslrootcert=$(pwd)/client-certs/ca.crt"
```

You should get a successful connection, proving the mTLS setup works for the admin.

### 4.2. Connecting as a New `appuser`

Finally, let's onboard a new, non-privileged application user and test its access.

**In PostgreSQL (connect as `admin`):**

```sql
-- Create the new role with the ability to log in
CREATE ROLE appuser LOGIN;

-- Make 'appuser' a member of the 'recipe_readers' group to inherit permissions
GRANT recipe_readers TO appuser;
```

**In your shell (with Vault):**

```bash
# Issue a client certificate for 'appuser'
vault write -format=json pki/issue/apprentice-role \
    common_name="appuser" > client-appuser-cert.json

# Extract the files
mkdir -p appuser-certs
jq -r .data.private_key < client-appuser-cert.json > appuser-certs/client-appuser.key
jq -r .data.certificate < client-appuser-cert.json > appuser-certs/client-appuser.crt
jq -r .data.issuing_ca < client-appuser-cert.json > appuser-certs/ca.crt
chmod 600 appuser-certs/client-appuser.key

# Connect as 'appuser' using its dedicated certificate
psql "host=localhost dbname=grand_grimoire user=appuser \
sslmode=verify-full \
sslcert=$(pwd)/appuser-certs/client-appuser.crt \
sslkey=$(pwd)/appuser-certs/client-appuser.key \
sslrootcert=$(pwd)/appuser-certs/ca.crt"
```
Success\! You are now connected as `appuser`, which can read the `recipes` table by inheriting permissions from its group role, all authenticated over a secure mTLS connection.

-----

## Update: Creating a Stable Network with Docker Compose

To create a more robust and production-like environment, it's better to give our PostgreSQL container a stable hostname. This is the key to issuing correct TLS certificates and makes it easier for other services to find the database.

### Step 1: Update Your `docker-compose.yml`

We will add a `hostname` (`grimoire`), define a custom `network`, and mount our configuration and certificate files directly as `volumes`. This avoids manually copying files into the container.

First, create a local folder named `postgres-config` and place your `postgresql.conf` and `pg_hba.conf` files inside it. Also, create a `server-certs` folder for the certificates we will generate.

```yaml
services:
  postgres:
    container_name: container-pg
    image: postgres
    hostname: grimoire    # <-- Give the container a stable hostname
    networks:             # <-- This connects it to the network for DNS
      - magic-net
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: root
      POSTGRES_DB: test_db
    volumes:
      - postgres-data:/var/lib/postgresql/data
      # Mount config files from your local machine
      - ./postgres-config/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./postgres-config/pg_hba.conf:/etc/postgresql/pg_hba.conf
      # Mount server certificates from your local machine
      - ./server-certs:/etc/postgresql/certs
    command: >
      -c 'config_file=/etc/postgresql/postgresql.conf'
      -c 'hba_file=/etc/postgresql/pg_hba.conf'
    restart: unless-stopped

  pgadmin:
    container_name: container-pgadmin
    image: dpage/pgadmin4
    depends_on:
      - postgres
    networks:
      - magic-net
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: root
    restart: unless-stopped

volumes:
  postgres-data:

# Define the custom network
networks:
  magic-net:
```

You must also update your `postgres-config/postgresql.conf` file to point to the new certificate location inside the container:

```ini
# Inside postgres-config/postgresql.conf
ssl_cert_file = '/etc/postgresql/certs/server.crt'
ssl_key_file = '/etc/postgresql/certs/server.key'
ssl_ca_file = '/etc/postgresql/certs/ca.crt'
```

### Step 2: Re-issue the Server Certificate for the New Hostname

Your server certificate must be valid for the new hostname, `grimoire`. First, update your Vault PKI role to allow `grimoire` as a valid domain.

```bash
vault write pki/roles/apprentice-role \
    allowed_domains="grimoire,academy.local,admin,appuser" \
    allow_subdomains=true \
    allow_bare_domains=true \
    max_ttl="1h"
```

Now, issue the new server certificate.

```bash
vault write -format=json pki/issue/apprentice-role \
    common_name="grimoire" \
    alt_names="localhost" \
    ip_sans="127.0.0.1" > postgres-cert.json

mkdir -p server-certs
jq -r .data.private_key < postgres-cert.json > server-certs/server.key
jq -r .data.certificate < postgres-cert.json > server-certs/server.crt
jq -r .data.issuing_ca < postgres-cert.json > server-certs/ca.crt
chmod 600 server-certs/server.key
```

With the new certificate in `./server-certs` and the updated `docker-compose.yml`, you can now run `docker-compose up -d --force-recreate`. Your PostgreSQL server will start with a stable hostname and the correct configuration.

### The Achievement: Why This is Better

  * **A Stable Identity for TLS** 🛡️: Your PostgreSQL server is now always known as `grimoire`. Because it has a stable name, you can issue a server certificate with `common_name="grimoire"`, which acts as its official, verifiable ID card.
  * **Correct Certificate Verification** ✅: Clients can now connect securely with `sslmode=verify-full`. When a client connects to `host=grimoire`, the server presents its certificate, and the names match. This crucial security check prevents man-in-the-middle attacks.

### Example: Onboarding a New `loguser`

This setup makes it easy to add new users.

**1. Create the Vault PKI Role**

```bash
vault write pki/roles/log-role \
    allowed_domains="loguser" \
    allow_bare_domains=true \
    max_ttl="15m"
```

**2. Issue the Certificate**

```bash
vault write -format=json pki/issue/log-role \
    common_name="loguser" \
    alt_names="localhost" \
    ip_sans="127.0.0.1" > loguser-certs.json

mkdir -p loguser-certs
jq -r .data.private_key < loguser-certs.json > loguser-certs/client-loguser.key
jq -r .data.certificate < loguser-certs.json > loguser-certs/client-loguser.crt
jq -r .data.issuing_ca  < loguser-certs.json > loguser-certs/ca.crt
chmod 600 loguser-certs/client-loguser.key
```

**3. Connect (after creating the `loguser` role in PostgreSQL)**

```bash
psql "host=localhost dbname=grand_grimoire user=loguser \
    sslmode=verify-full \
    sslcert=$(pwd)/loguser-certs/client-loguser.crt \
    sslkey=$(pwd)/loguser-certs/client-loguser.key \
    sslrootcert=$(pwd)/loguser-certs/ca.crt"
```
