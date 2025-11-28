
# OpenSSL Cheat Sheet 😎

> A practical, no-nonsense guide from beginner to advanced.  
> All commands assume `openssl` is installed and in your `$PATH`.

---

## 0. Quick Concepts (Read This First)

- **Key** = secret or public data used to encrypt/sign things.
- **Private key** = keep it safe, never share.
- **Public key** = can be shared with others.
- **CSR (Certificate Signing Request)** = “Hey CA, please sign this public key for this domain.”
- **Certificate (`.crt`, `.pem`, `.cer`)** = public key + identity info + signature.
- **CA (Certificate Authority)** = entity that signs certs (root or intermediate).
## 📚 Table of Contents

- [0. Quick Concepts](#0-quick-concepts-read-this-first)
- [1. Check OpenSSL Version](#1-check-openssl-version)
- [2. Generate Private Keys](#2-generate-private-keys)
  - [2.1 RSA Private Key](#21-rsa-private-key)
  - [2.2 RSA with genpkey](#22-rsa-private-key-modern-genpkey)
  - [2.3 EC Private Key](#23-ec-elliptic-curve-private-key)
  - [2.4 Add / Remove Password](#24-convert-key-with-or-without-password)
- [3. Create a CSR](#3-create-a-csr-certificate-signing-request)
  - [3.1 Interactive CSR](#31-simple-csr-interactive-questions)
  - [3.2 Non-Interactive CSR](#32-non-interactive-csr-using-a-config-file)
- [4. Self-Signed Certificates](#4-self-signed-certificates-for-testing)
- [5. View and Inspect Files](#5-view-and-inspect-files)
- [6. File Formats: PEM, DER, PFX](#6-file-formats-pem-der-pfx-etc)
- [7. Hashing & Checksums](#7-hashing--checksums)
- [8. Symmetric Encryption (AES)](#8-symmetric-encryption-aes-etc)
- [9. Asymmetric Crypto: Sign & Verify](#9-asymmetric-crypto-sign--verify)
- [10. Certificate Chain & Expiry](#10-check-certificate-chains--expiry)
- [11. Debugging TLS Connections](#11-debugging-tls--https-connections)
- [12. Diffie-Hellman Parameters](#12-generate-diffie-hellman-parameters)
- [13. Password & Random Tools](#13-password--random-utilities)
- [14. Identify Unknown Files](#14-mini-what-the-heck-is-this-file-guide)
- [15. Useful One-Liners](#15-common-one-liners)
- [16. Security Best Practices](#16-tips--best-practices)
- [17. Quick Reference Table](#17-quick-reference-table)


Common file formats:

- `.pem` = Base64 text with `-----BEGIN ...` / `-----END ...`
- `.key` = private key (usually PEM)
- `.crt` / `.cer` = certificate (PEM or DER)
- `.pfx` / `.p12` = PKCS#12 bundle (cert + key + chain), usually password protected

---

## 1. Check OpenSSL Version

```bash
openssl version
openssl version -a   # with build info
````

---

## 2. Generate Private Keys

### 2.1 RSA Private Key

```bash
# 2048-bit RSA key (enough for most uses)
openssl genrsa -out server.key 2048

# 4096-bit (stronger but heavier)
openssl genrsa -out server-4096.key 4096
```

### 2.2 RSA Private Key (Modern `genpkey`)

```bash
openssl genpkey -algorithm RSA -out server.key -pkeyopt rsa_keygen_bits:2048
```

### 2.3 EC (Elliptic Curve) Private Key

```bash
# List available curves
openssl ecparam -list_curves

# Generate EC key using prime256v1 (aka secp256r1)
openssl ecparam -name prime256v1 -genkey -noout -out server-ec.key
```

### 2.4 Convert Key WITH or WITHOUT Password

```bash
# Add password to private key (PEM)
openssl rsa -aes256 -in server.key -out server-protected.key

# Remove password (be careful!)
openssl rsa -in server-protected.key -out server-nopass.key
```

---

## 3. Create a CSR (Certificate Signing Request)

### 3.1 Simple CSR (Interactive Questions)

```bash
openssl req -new -key server.key -out server.csr
```

You’ll be asked for:

* Country Name (C)
* State or Province (ST)
* Locality (L)
* Organization (O)
* Organizational Unit (OU)
* Common Name (CN)  → usually your domain, e.g. `example.com`
* Email Address

### 3.2 Non-Interactive CSR (using a config file)

`openssl.cnf` example snippet:

```ini
[ req ]
default_bits       = 2048
default_md         = sha256
prompt             = no
distinguished_name = dn
req_extensions     = req_ext

[ dn ]
C  = US
ST = Some-State
L  = Some-City
O  = My Company
OU = IT
CN = example.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = example.com
DNS.2 = www.example.com
```

Generate CSR:

```bash
openssl req -new -key server.key -out server.csr -config openssl.cnf
```

---

## 4. Self-Signed Certificates (for testing)

### 4.1 Quick Self-Signed Cert (No CSR file)

```bash
openssl req -x509 -newkey rsa:2048 -keyout server.key -out server.crt \
  -days 365 -nodes
```

* `-x509`: make cert, not CSR
* `-newkey rsa:2048`: create key + cert in one go
* `-nodes`: do NOT encrypt the private key (no password)

### 4.2 Self-Sign From Existing Key + CSR

```bash
openssl x509 -req -in server.csr -signkey server.key -out server.crt -days 365
```

---

## 5. View and Inspect Files

### 5.1 View Certificate (Human-Readable)

```bash
openssl x509 -in server.crt -noout -text
```

### 5.2 View CSR

```bash
openssl req -in server.csr -noout -text
```

### 5.3 View Private Key Details

```bash
# RSA key
openssl rsa -in server.key -check -noout

# EC key
openssl ec -in server-ec.key -check -noout
```

### 5.4 Extract Public Key from Private Key or Certificate

```bash
# From private key
openssl pkey -in server.key -pubout -out server.pub

# From certificate
openssl x509 -in server.crt -pubkey -noout > server.pub
```

---

## 6. File Formats: PEM, DER, PFX, etc.

### 6.1 PEM ⇔ DER

```bash
# PEM cert -> DER
openssl x509 -in server.crt -outform der -out server.der

# DER cert -> PEM
openssl x509 -in server.der -inform der -out server.pem
```

### 6.2 Create PKCS#12 (.pfx / .p12)

```bash
# Key + cert -> PFX
openssl pkcs12 -export -out server.pfx -inkey server.key -in server.crt

# Key + cert + chain -> PFX
openssl pkcs12 -export -out server.pfx \
  -inkey server.key \
  -in server.crt \
  -certfile chain.crt
```

### 6.3 Extract from PKCS#12

```bash
# Extract private key
openssl pkcs12 -in server.pfx -nocerts -out server.key

# Extract certificate
openssl pkcs12 -in server.pfx -clcerts -nokeys -out server.crt

# Extract everything
openssl pkcs12 -in server.pfx -nodes -out all.pem
```

---

## 7. Hashing & Checksums

### 7.1 Hash a File

```bash
openssl dgst -md5 file.txt
openssl dgst -sha1 file.txt
openssl dgst -sha256 file.txt
openssl dgst -sha512 file.txt
```

### 7.2 Hash with HMAC (Keyed Hash)

```bash
openssl dgst -sha256 -hmac "secretkey" file.txt
```

---

## 8. Symmetric Encryption (AES, etc.)

> Note: built-in `enc` is ok for quick stuff, but for serious security use modern tools/libs.

### 8.1 Encrypt a File (Password-Based)

```bash
# AES-256-CBC encryption
openssl enc -aes-256-cbc -salt -in plain.txt -out encrypted.bin
```

You’ll be asked for a password.

### 8.2 Decrypt a File

```bash
openssl enc -d -aes-256-cbc -in encrypted.bin -out decrypted.txt
```

### 8.3 Encrypt/Decrypt with Base64 Output

```bash
# Encrypt and output Base64
openssl enc -aes-256-cbc -salt -in plain.txt -out encrypted.b64 -base64

# Decrypt from Base64
openssl enc -d -aes-256-cbc -in encrypted.b64 -out decrypted.txt -base64
```

---

## 9. Asymmetric Crypto: Sign & Verify

### 9.1 Sign a File (Detached Signature)

```bash
# Create signature.sig using private key
openssl dgst -sha256 -sign private.key -out file.sig file.txt
```

### 9.2 Verify Signature

```bash
# Verify with public key
openssl dgst -sha256 -verify public.pem -signature file.sig file.txt
```

(Exit code `0` = success, non-zero = failure.)

---

## 10. Check Certificate Chains & Expiry

### 10.1 Basic Certificate Info

```bash
openssl x509 -in server.crt -noout -subject -issuer -dates
```

### 10.2 Verify a Cert Against a CA

```bash
openssl verify -CAfile ca.crt server.crt
```

### 10.3 Verify with Intermediate Chain

```bash
# ca-chain.crt contains intermediate + root
openssl verify -CAfile ca-chain.crt server.crt
```

---

## 11. Debugging TLS / HTTPS Connections

### 11.1 Connect to a TLS Server (s_client)

```bash
openssl s_client -connect example.com:443
```

### 11.2 Show Server Certificate

```bash
openssl s_client -connect example.com:443 -showcerts
```

### 11.3 Check a Specific SNI / Hostname

```bash
openssl s_client -connect example.com:443 -servername example.com
```

### 11.4 Test SMTP over STARTTLS (Example: port 587)

```bash
openssl s_client -starttls smtp -connect mail.example.com:587
```

Other `-starttls` options: `http`, `imap`, `pop3`, `ftp`, etc.

---

## 12. Generate Diffie-Hellman Parameters

(Used in some older configs; modern setups often use ECDHE instead.)

```bash
openssl dhparam -out dhparam.pem 2048
```

---

## 13. Password & Random Utilities

### 13.1 Generate Random Bytes

```bash
# 16 random bytes as hex
openssl rand -hex 16

# 32 random bytes, base64
openssl rand -base64 32
```

### 13.2 Generate Password Hash (for /etc/shadow style)

```bash
# SHA-512 based password hash (interactive)
openssl passwd -6
```

Common switches:

* `-1` = MD5 (old, don’t use)
* `-5` = SHA-256
* `-6` = SHA-512

---

## 14. Mini “What the heck is this file?” Guide

```bash
# Try as certificate
openssl x509 -in unknown-file -noout -text

# Try as CSR
openssl req -in unknown-file -noout -text

# Try as private key
openssl rsa -in unknown-file -check -noout    # RSA
openssl ec  -in unknown-file -check -noout    # EC

# Try to see if it's a PKCS#12 bundle
openssl pkcs12 -in unknown-file -info
```

If one of those works → you know what you have 🙂

---

## 15. Common One-Liners

### 15.1 Quickly Check Certificate Expiry of a Site

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -dates
```

### 15.2 Get Only the CN (Common Name)

```bash
openssl x509 -in server.crt -noout -subject
# then look for CN=
```

### 15.3 Convert NGINX/Apache Style Chain

```bash
# Combine leaf + intermediate into one chain file
cat server.crt intermediate.crt > fullchain.crt
```

---

## 16. Tips & Best Practices

* Use at least **RSA 2048** or **EC prime256v1** for new keys.
* Prefer **SHA-256** over older hashes like SHA-1 or MD5.
* Protect private keys:

  * File permissions: `chmod 600 server.key`
  * Store in safe location (and backups).
* For production certificates:

  * Always include the **full chain** (leaf + intermediate).
  * Regularly check expiry dates (monitoring/alerts).

---

## 17. Quick Reference Table

| Task                     | Command (short version)                                                             |
| ------------------------ | ----------------------------------------------------------------------------------- |
| Generate RSA key         | `openssl genrsa -out key.pem 2048`                                                  |
| Generate EC key          | `openssl ecparam -name prime256v1 -genkey -out key.pem`                             |
| Create CSR               | `openssl req -new -key key.pem -out req.csr`                                        |
| Self-signed cert         | `openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes` |
| View cert                | `openssl x509 -in cert.pem -noout -text`                                            |
| View CSR                 | `openssl req -in req.csr -noout -text`                                              |
| Cert → DER               | `openssl x509 -in cert.pem -outform der -out cert.der`                              |
| Make PFX                 | `openssl pkcs12 -export -out cert.pfx -inkey key.pem -in cert.pem`                  |
| Verify cert              | `openssl verify -CAfile ca.pem cert.pem`                                            |
| Hash file (SHA-256)      | `openssl dgst -sha256 file`                                                         |
| Random 32 bytes (base64) | `openssl rand -base64 32`                                                           |
| Connect to HTTPS         | `openssl s_client -connect example.com:443`                                         |

---

*Feel free to copy this file as `OPENSSL-CHEATSHEET.md` into your GitHub repo.*

```

::contentReference[oaicite:0]{index=0}
```
