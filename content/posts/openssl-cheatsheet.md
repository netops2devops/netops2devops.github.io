---
title: OpenSSL quick cheatsheet
date: 2025-03-16
tags: ["Security", "PKI"]
author: "Kapil Agrawal"
comments: false
---

Every now and then I come across a situation where I need to work with PKI, X.509 certs etc. specially after transtioning into a more security focused role. I wanted to document some super handy OpenSSL one liners which I often rely on.

### Generate (unencrypted) private key

```
❯ openssl genpkey -algorithm rsa -out priv.key
```

### Generate (Encrypted) private key

```
# Let's review all supported cipher options first
❯ openssl list -cipher-algorithms

❯ openssl genpkey -algorithm rsa -out priv.key -AES128
.+......+..+...............+......+......+.........+..........+..............+.+.................+............+...+...+....+..+.+........+..........+......+......+........+......+.........+..........+..............+.......+...+...+...+......+.....+..........+........+...+...+............+....+.........++++++
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
```

### Extract public key from a private key

```
❯ openssl pkey -in priv.key -pubout -out pub.key
Enter pass phrase for priv.key:
```

We now have our key pairs in PEM format

### Using generated keys for SSH

```
# add key to ssh-agent
❯ ssh-add priv.key

# show ssh public key in PKCS8 format
❯ ssh-keygen -f pub.key -i -mPKCS8
OR
❯ ssh-add -L
```

### Decoding keys with OpenSSL

```
# keys must be in PEM format
❯ openssl pkey -in priv.key -noout -text

# decode a public key (RSA)
❯ openssl rsa -RSAPublicKey_in -in pub.key -noout -text
```

### Create a new CSR

```
# use a pre-existing private key
❯ openssl req -new -key priv.key -out csr.pem

# Generates a fresh new private key for CSR
❯ openssl req -new -out csr.pem

# decode a CSR
❯ openssl req -in csr.pem -noout -text
```

### Create a x509 cert

```
# Only using a private key
❯ openssl req -x509 -key priv.key -out cert.pem

# Using a CSR & private key
❯ openssl x509 -req -in csr.pem -key priv.key -out cert.pem

# Decoding a certificate
❯ openssl x509 -in cert.pem -noout -text
```

### Encrypting & Decrypting files

```
# encrypt with public key
❯ openssl pkeyutl -encrypt -in file.txt -out encrypted.txt -pubin -inkey pub.key

# decrypt using private key
❯ openssl pkeyutl -decrypt -in encrypted.txt -out decrypted.txt -inkey priv.key

# Verify file integrity
❯ shasum file.txt
❯ shasum decrypted.txt
```
