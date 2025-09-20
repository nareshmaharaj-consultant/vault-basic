# Vault Demo: Dynamic Secrets & PKI with PostgreSQL 🧙‍♂️

Welcome to the Alchemist's Vault! This repository contains a series of hands-on tutorials that walk you through some of the most powerful features of HashiCorp Vault, all told through the story of a magical school, its secret-filled database, and an apprentice named Pip.

The goal is to provide practical, step-by-step examples of how to solve real-world security challenges using Vault's core engines.

---
## Tutorial Guides

Follow these guides in order to learn how to manage database credentials dynamically, perform encryption as a service, and establish a secure, certificate-based mTLS connection to PostgreSQL.

* **[Part 1: Dynamic Database Secrets](./part1.md)**
    * Learn how to configure Vault to dynamically generate temporary, on-demand credentials for a PostgreSQL database, eliminating the need for long-lived static passwords.

* **[Part 2: Encryption as a Service](./02-encryption-as-a-service.md)**
    * Discover how to use Vault's Transit Secrets Engine to encrypt and decrypt sensitive data without ever exposing the encryption keys to your application.

* **[Part 3: PKI and Mutual TLS (mTLS)](./03-pki-and-mtls.md)**
    * Set up Vault as a Certificate Authority (CA) to issue TLS certificates for your PostgreSQL server and clients, enabling a secure, passwordless, and verifiable mTLS connection.

---
### Prerequisites

To follow along with these tutorials, you will need the following tools installed on your machine:
* Docker & Docker Compose
* HashiCorp Vault CLI
* PostgreSQL Client (`psql`)
* `jq` (a command-line JSON processor)
