# The Alchemist's Vault: A Magical Guide to Dynamic Secrets and Encryption

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
