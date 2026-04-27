## Introduction

The contents of this repo provide explanation of the demo part of an introductory level talk on Public Key Infrastructure.  I wanted to demo, in a simple way, creating private/public key pairs, a CSR and a simple case of where a CA would fit in.

In 2026, it is impossible to ignore AI.  The repo includes ```prompt.txt``` which was used to generate everything below this introduction.  It is as the AI gave it to me with some very minor tweaks.

The talk being _introductory level_:  as mentioned in the slides (link to follow) Dunning-Kruger applies in the PKI space.  Running PKI at scale is the work of a smart team, not a part time individual.  This will become even more evident as both certificate life spans and domain validation times shorten over the coming years.

So, this is part of an _introductory level_ talk on PKI.  The focus is somewhat self-centred - the things that I struggled to understand when I was first creating CSRs as a sysadmin and the questions I get asked now.  (And no, I don't consider myself an SME in this are - that would be prime Dunning-Kruger!)


***

````markdown
# OpenSSL Certificate Authority (CA) and Certificate Signing Walkthrough

## Overview

This walkthrough demonstrates a **basic public key infrastructure (PKI) workflow** using OpenSSL.  
It simulates:

- A **Certificate Authority (CA)** that issues digital certificates
- A **customer/server** that requests a certificate by generating a **Certificate Signing Request (CSR)**

The process mirrors how TLS/SSL certificates are typically created and signed in real environments, but in a simplified, local setup.

---

## Directory Structure

Two directories are created to represent the two roles involved:

- `CA/` — acts as the Certificate Authority
- `customer/` — acts as the certificate requester (e.g., a server)

```text
.
├── CA/
└── customer/
````

***

## Step 1: Create Working Directories

```bash
mkdir customer CA
```

**Explanation:**

*   Creates two directories:
    *   `customer`: where the private key and CSR are generated
    *   `CA`: where the CA’s private key and certificate live and where CSRs are signed

***

## Step 2: Set Up the Certificate Authority (CA)

Move into the CA directory:

```bash
cd CA
```

### 2.1 Generate the CA Private Key

```bash
openssl genpkey -algorithm RSA -out CA.key -pkeyopt rsa_keygen_bits:4096
```

**What this does:**

*   Generates a **4096‑bit RSA private key**
*   This key is the **root of trust** for the CA
*   Used later to **cryptographically sign client certificates**

**Key points:**

*   `genpkey` is OpenSSL’s modern key generation command
*   `-algorithm RSA` specifies RSA
*   `rsa_keygen_bits:4096` selects strong key length
*   Output file: `CA.key` (must be kept secret)

***

### 2.2 Create a Self‑Signed CA Certificate

```bash
openssl req -key CA.key -new -x509 -days 365 -out CA.crt
```

**What this does:**

*   Creates a **self‑signed X.509 certificate**
*   Uses the CA’s private key (`CA.key`)
*   Valid for **365 days**

**Why self‑signed?**

*   Root CAs are self‑signed because they are their own trust anchor
*   In real environments, this certificate would be installed into trust stores

**Important options:**

*   `-req` — certificate request functionality
*   `-new` — generate a new certificate
*   `-x509` — output a certificate instead of a CSR
*   `-days 365` — validity period
*   `-out CA.crt` — public CA certificate

You will be prompted for certificate fields such as:

*   Country
*   Organization
*   Common Name (CN)

***

## Step 3: Customer Generates a Key Pair and CSR

Move to the customer directory:

```bash
cd ../customer
```

### 3.1 Generate the Customer Private Key

```bash
openssl genpkey -algorithm RSA -out private.key -pkeyopt rsa_keygen_bits:4096
```

**What this does:**

*   Creates a **4096‑bit RSA private key** for the customer/server
*   This key will later be used for:
    *   TLS handshakes
    *   Proving identity via the signed certificate

**Security note:**  
This private key must remain confidential.

***

### 3.2 Derive the Public Key

```bash
openssl pkey -in private.key -pubout -out public.key
```

**What this does:**

*   Extracts the **public key** from the private key
*   Writes it to `public.key`

**Why this matters:**

*   Demonstrates the asymmetric key relationship
*   The public key is included in the CSR and final certificate

***

### 3.3 Create a Certificate Signing Request (CSR)

```bash
openssl req -new -key private.key -out request.csr
```

**What this does:**

*   Creates a **CSR** containing:
    *   The customer’s public key
    *   Subject information (CN, O, etc.)
*   The CSR is **signed with the customer’s private key**, proving key ownership

**Important options:**

*   `-new` — create a new request
*   `-key private.key` — sign the request with this key
*   `-out request.csr` — output CSR file

You will again be prompted for subject details, most importantly:

*   **Common Name (CN)** — typically the server’s DNS name

***

## Step 4: Send the CSR to the CA

```bash
cp request.csr ../CA
```

**What this represents:**

*   Simulates sending the CSR to a Certificate Authority
*   In real life, this would be uploaded to a CA or sent via an API

***

## Step 5: CA Signs the CSR

Move back to the CA directory:

```bash
cd ../CA
```

### 5.1 Generate a Signed Certificate

```bash
openssl x509 -req -in request.csr -CA CA.crt -CAkey CA.key -CAcreateserial -out server.crt -days 365 -sha256
```

**What this does:**

*   Verifies the CSR
*   Issues a **signed certificate** using the CA’s key
*   Produces `server.crt`, valid for 365 days

**Important options explained:**

*   `-req` — input is a CSR
*   `-in request.csr` — CSR to sign
*   `-CA CA.crt` — CA certificate used for signing
*   `-CAkey CA.key` — CA private key
*   `-CAcreateserial` — creates a serial number file if missing
*   `-out server.crt` — output signed certificate
*   `-days 365` — certificate lifetime
*   `-sha256` — strong hashing algorithm

**Result:**

*   `server.crt` now chains back to `CA.crt`
*   Clients that trust `CA.crt` will trust `server.crt`

***

## Step 6: Return the Certificate to the Customer

```text
server.crt → customer
```

**Final state (customer side):**

*   `private.key` — private key
*   `server.crt` — CA‑signed certificate
*   `CA.crt` — CA certificate (used as a trust anchor)

These files together enable:

*   TLS server authentication
*   Secure encrypted communications

***

## Summary

This process demonstrates:

1.  CA key and self‑signed root certificate creation
2.  Customer key pair generation
3.  CSR creation and submission
4.  CA validation and certificate issuance

While simplified, this workflow closely mirrors real‑world certificate issuance and is ideal for learning, testing, or internal PKI labs.

***

## Notes

*   Real CAs enforce validation checks before signing
*   Real deployments use certificate extensions (`openssl.cnf`)
*   Private keys should be encrypted and access‑controlled

