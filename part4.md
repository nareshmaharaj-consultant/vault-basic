# The Alchemist's Vault, Part 4: Pip's New Spell - A Secure Python API with TDD

Welcome back to the Grand Academy of Arcane Arts! In our previous lessons, we fortified the Grand Grimoire (our PostgreSQL database) to require secure mTLS connections. Now, Headmaster Alistair's final edict is in: Apprentice Pip must create a new spell—a Python REST API—that can dynamically generate its own certificate from the Alchemist's Vault to securely read recipes.

In this final chapter, we'll build this application using a **Test-Driven Development (TDD)** approach. We will first write tests that define *what* our magical logic should do, and then write the code to make those tests pass.

### Part 4: Pip's New Spell - The Python Flask API 🤖

Apprentice Pip can now write a new spell—a Python API that gets a certificate from Vault on-demand to securely query the database.

### 1. Preparing Pip's Spellbook (Project Setup)

Before we write any code, we need to gather our magical libraries. These are the building blocks of our application.

Install the required libraries using pip. It's best practice to do this inside a virtual environment.
```bash
pip install flask hvac psycopg2-binary requests
````

  * `flask`: A lightweight framework for creating our web server.
  * `hvac`: The official HashiCorp Vault client for Python.
  * `psycopg2-binary`: The most popular PostgreSQL adapter for Python.

### 2\. The Headmaster's Secret Incantations (Environment Variables)

To connect to Vault, our application needs a few secrets: the Vault server's address, a valid token, and the CA certificate to verify its TLS connection. We must **never** hardcode these in our script. Instead, we'll use environment variables.

You must set these in your IDE's run configuration. For example, in IntelliJ/PyCharm:

1.  Go to **Run -\> Edit Configurations...**.
2.  Find or create a run configuration for your Python script.
3.  In the **Environment variables** field, add the following key-value pairs.

<!-- end list -->

  * `VAULT_ADDR`: The address of your Vault server (e.g., `https://127.0.0.1:8200`).
  * `VAULT_TOKEN`: The root token from your `vault server -dev` instance.
  * `VAULT_CACERT`: The absolute path to the CA certificate used by your Vault dev server.

### 3\. Forging the Magical Core with TDD (The `api.py` Logic)

With TDD, we first define our expectations in a test file. This file contains all the core, reusable logic for interacting with Vault and PostgreSQL. It's the engine of our application, and the tests ensure it's reliable.

**`api.py`**

```python
import enum
import unittest
import os
import hvac
import tempfile
import psycopg2

class ApiError(Exception): pass
class ApiErrorEnvironmentSettings(ApiError): pass
class ApiErrorDBConnect(ApiError): pass

class ApiSslEnum(enum.Enum):
    SSL_ROOT_CA_CERTIFICATE = '"ssl_root_ca'
    SSL_CERTIFICATE = 'ssl_cert'
    SSL_KEY = 'ssl_key'

def get_env_setting(env_setting):
    if os.environ.get(env_setting) is None:
        raise ApiErrorEnvironmentSettings( f"{env_setting} has not been set" )
    return os.environ.get( env_setting)

class Api:
    def __init__(self):
        self.harness_active = True
        self.vault_client = None
        self.cert_files = {}
        self.connection = None

    def init_vault_client(self, address, token, ca_cert):
        client = hvac.Client(url=address, token=token, verify=ca_cert)
        self.vault_client = client
        return client is not None

    def get_vault_pki_certificate(self, common_name, mount, alt_names, ip_sans):
        pki_response = self.vault_client.secrets.pki.generate_certificate(
            name='apprentice-role',
            common_name=common_name,
            mount_point=mount,
            extra_params=dict( alt_names=alt_names, ip_sans=ip_sans )
        )
        return pki_response

    def read_vault_pki_certificate(self, pki_serial_number):
        certificate = self.vault_client.secrets.pki.read_certificate(pki_serial_number)
        return certificate

    def write_vault_pki_certificate_response_to_temp_files(self, pki_response):
        with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.crt') as ca_file:
            ca_file.write( pki_response['data']['issuing_ca'] )
            self.cert_files[ApiSslEnum.SSL_ROOT_CA_CERTIFICATE] = ca_file.name

        with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.crt') as cert_file:
            cert_file.write( pki_response['data']['certificate'] )
            self.cert_files[ApiSslEnum.SSL_CERTIFICATE] = cert_file.name

        with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.key') as private_key_file:
            private_key_file.write( pki_response['data']['private_key'] )
            self.cert_files[ApiSslEnum.SSL_KEY] = private_key_file.name

        return self.cert_files

    def create_db_connection(self, address, port, database, user, ssl_root_ca, ssl_cert, ssl_key):
        try:
            conn_string = f"host='{address}' dbname='{database}' user='{user}' " \
                          f"sslmode='verify-full' sslrootcert='{ssl_root_ca}' " \
                          f"sslcert='{ssl_cert}' sslkey='{ssl_key}'"
            connection = psycopg2.connect(conn_string)
            self.connection = connection
            return self.connection
        except:
            raise ApiErrorDBConnect(f"Unable to connect to database '{database}'")


# Set env variable in Run Configurations
class ApiTest(unittest.TestCase):
    def setUp(self):
        self.api = Api()

        self.api.init_vault_client(
            get_env_setting("VAULT_ADDR"),
            get_env_setting("VAULT_TOKEN"),
            get_env_setting("VAULT_CACERT"),
        )

        self.vault_client = self.api.vault_client

        self.pki_response = self.api.get_vault_pki_certificate(
            "pip-script.academy.local",
            "pki",
            "localhost,grimoire",
            "127.0.0.1"
        )

    def test_suite(self):
        self.assertTrue(self.api.harness_active)

    def test_vault_env_variable_not_set(self):
        with self.assertRaises(ApiErrorEnvironmentSettings):
            get_env_setting("X_VAULT_ADDR")

    def test_vault_env_variable_set(self):
        vault_expected_host = "[https://127.0.0.1:8200](https://127.0.0.1:8200)"
        self.assertEqual( vault_expected_host, get_env_setting("VAULT_ADDR") )

        vault_token_begins_with = "hvs."
        self.assertTrue( vault_token_begins_with,  get_env_setting("VAULT_TOKEN").startswith("hvs.") )

    def test_initialise_vault_client(self):
        self.assertTrue( self.api.init_vault_client(
            get_env_setting("VAULT_ADDR"),
            get_env_setting("VAULT_TOKEN"),
            get_env_setting("VAULT_CACERT"),
        ) )

    def test_generate_pki_certificate(self):
        self.assertTrue( len( self.pki_response[ "data" ][ "serial_number" ] ) > 0  )

    def test_read_pki_certificate(self):
        serial_number = self.pki_response[ "data" ][ "serial_number" ]
        certificate = self.api.read_vault_pki_certificate(serial_number)
        self.assertTrue( len( certificate[ "data" ][ "certificate" ] ) > 0 )

    def test_write_pki_response_to_temp_files(self):
        files = self.api.write_vault_pki_certificate_response_to_temp_files(self.pki_response)
        self.assertIn( ApiSslEnum.SSL_ROOT_CA_CERTIFICATE, files )
        self.assertIn( ApiSslEnum.SSL_CERTIFICATE, files )
        self.assertIn( ApiSslEnum.SSL_KEY, files )

    def test_connection_to_postgres(self):
        files = self.api.write_vault_pki_certificate_response_to_temp_files(self.pki_response)
        self.assertTrue( True, self.api.create_db_connection(
            "localhost",
            5432,
            "grand_grimoire",
            "pip-script.academy.local",
            files[ApiSslEnum.SSL_ROOT_CA_CERTIFICATE],
            files[ApiSslEnum.SSL_CERTIFICATE],
            files[ApiSslEnum.SSL_KEY] ).status
        )

    def test_read_pki_certificate_with_serial_number(self):
        serial_number = "67:e2:c1:e8:1c:10:01:a4:57:3d:da:8a:9d:3f:1e:99:51:41:67:52"
        certificate = self.api.read_vault_pki_certificate(serial_number)
        self.assertTrue( len( certificate[ "data" ][ "certificate" ] ) > 0 )

if __name__ == "__main__":
    unittest.main()
```

### 4\. Casting the Spell (The Flask Web Application)

Now we create the web application that uses our tested API logic. This file, `app.py`, is the lightweight wrapper that exposes our magic to the world as a REST endpoint.

**Note:** Flask's built-in server is great for development, but for a real-world application, you would run it using a production-grade WSGI server like Gunicorn or uWSGI.

**`app.py`**

```python
import os

from api import Api, get_env_setting, ApiSslEnum
from flask import Flask, jsonify

# --- Flask App ---
app = Flask(__name__)
@app.route("/get-recipe/<int:recipe_id>")
def get_recipe(recipe_id):

    files = None

    try:
        api = Api()

        api.init_vault_client(
            get_env_setting("VAULT_ADDR"),
            get_env_setting("VAULT_TOKEN"),
            get_env_setting("VAULT_CACERT")
        )

        pki_response = api.get_vault_pki_certificate(
            "pip-script.academy.local",
            "pki",
            "localhost,grimoire",
            "127.0.0.1"
        )

        files = api.write_vault_pki_certificate_response_to_temp_files( pki_response )

        connection = api.create_db_connection(
            "localhost",
            5432,
            "grand_grimoire",
            "pip-script.academy.local",
            files[ApiSslEnum.SSL_ROOT_CA_CERTIFICATE],
            files[ApiSslEnum.SSL_CERTIFICATE],
            files[ApiSslEnum.SSL_KEY]
        )

        with connection.cursor() as cur:
            print("Successfully connected to PostgreSQL with TLS certificate!")
            cur.execute( "SELECT name, ingredients FROM recipes WHERE id = %s", (recipe_id,) )
            recipe = cur.fetchone()
            if recipe:
                return jsonify({"name": recipe[0], "ingredients": recipe[1]})
            else:
                return jsonify({"error": "Recipe not found"}), 404

    except Exception as e:
        print(f"An error occurred: {e}")
        return jsonify({"error": str(e)}), 500

    finally:
        # 5. Clean up temporary certificate files
        if files:
            for f in files.values():
                if os.path.exists(f):
                    os.remove(f)
            print("Cleaned up temporary certificate files.")

if __name__ == '__main__':
    app.run(debug=True, port=5001)
```

### 5\. The Grand Finale - Running the API

Before running the API, we need to create a PostgreSQL role for it to use. The role name must match the Common Name in the certificate we generate.

**In PostgreSQL (connect as `admin`):**

```sql
CREATE ROLE "pip-script.academy.local" LOGIN;
GRANT recipe_readers TO "pip-script.academy.local";
```

Now, run the Flask application:

```bash
python app.py
```

Finally, test the endpoint from another terminal:

```bash
curl http://127.0.0.1:5001/get-recipe/1
```

You should see the recipe returned, all over a secure mTLS connection\!

```json
{
  "ingredients": "Moon dust, shadow essence",
  "name": "Elixir of Invisibility"
}
```

### The Spell is Complete

Congratulations\! You have successfully built a zero-trust system where an application generates its own short-lived, unique cryptographic identity from Vault at runtime to securely authenticate to a database. You've mastered some of the most powerful and practical spells in the security wizard's grimoire.
