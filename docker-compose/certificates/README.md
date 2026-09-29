# gRPC TLS certificates

This directory contains the single certificate set shared by all version-specific `docker-compose-oauth2-tls.yaml` configurations. Generate or replace certificates here once; do not create separate certificate sets under individual version directories.

The TLS setup secures the Orchestration gRPC endpoint on port 26500 only. REST/UI on port 8080 and Keycloak on port 18080 remain HTTP.

The tracked templates have separate responsibilities:

- `openssl-root-ca.cnf` creates a local development root CA that Electron, Node.js, Java, and Go clients can use as a trust anchor.
- `openssl-san.cnf` contains the subject and requested extensions used by the Camunda operator to generate a CSR or local self-signed certificate.
- `openssl-signing-extensions.cnf` is an optional CA policy template used only when the CA administrator needs to override the extensions requested in the CSR.

Update `openssl-san.cnf` if the deployment uses hostnames or IP addresses other than `orchestration`, `localhost`, and `127.0.0.1`. When the optional CA policy template is used, its SAN values are intentionally defined separately so the CA can control the issued certificate independently.

## Option 1: certificate signed by a local development root CA

Run these commands from the repository's `docker-compose/` directory, which contains this `certificates/` directory:

```bash
openssl req -x509 -newkey rsa:3072 -sha256 -days 3650 -nodes \
  -keyout certificates/root-ca.key \
  -out certificates/root-ca.crt \
  -config certificates/openssl-root-ca.cnf

openssl req -new -newkey rsa:2048 -sha256 -nodes \
  -keyout certificates/tls.key \
  -out certificates/orchestration.csr \
  -config certificates/openssl-san.cnf

openssl x509 -req \
  -in certificates/orchestration.csr \
  -CA certificates/root-ca.crt \
  -CAkey certificates/root-ca.key \
  -CAcreateserial \
  -out certificates/tls.crt \
  -days 825 -sha256 \
  -copy_extensions copy

cp certificates/root-ca.crt certificates/ca.crt
chmod 600 certificates/root-ca.key certificates/tls.key
chmod 644 certificates/tls.crt certificates/ca.crt
```

The root CA is self-signed with `CA:TRUE`; the gRPC server certificate is a separate `CA:FALSE` leaf signed by that root. `ca.crt` contains only the public root CA and is used by Connectors, Desktop Modeler, and other clients as their trust anchor. Do not use a self-signed `CA:FALSE` server certificate as `ca.crt`: some clients such as `zbctl` may accept it, while Electron/OpenSSL can reject it with `unable to verify the first certificate`.

The generated certificates, CSR, serial file, and private keys are ignored by Git. They are intended for local development only. Never distribute `root-ca.key` or `tls.key` to clients.

## Option 2: certificate signed by an internal or corporate CA

### Generate and inspect the CSR

Generate the private key and Certificate Signing Request (CSR):

```bash
openssl req -new -newkey rsa:2048 -sha256 -nodes \
  -keyout certificates/tls.key \
  -out certificates/orchestration.csr \
  -config certificates/openssl-san.cnf
chmod 600 certificates/tls.key
```

Inspect the CSR before sending it to the CA:

```bash
openssl req -in certificates/orchestration.csr \
  -noout -verify -subject -text
```

Ask the CA administrator to issue a TLS server certificate with these extensions:

- `subjectAltName`: `DNS:orchestration`, `DNS:localhost`, and `IP:127.0.0.1`
- `keyUsage`: `digitalSignature`, `keyEncipherment`
- `extendedKeyUsage`: `serverAuth`

The CA may not automatically copy extensions from the CSR. Verify that the issued leaf certificate contains the requested SAN and Extended Key Usage before using it.

### Sign the CSR with an existing root CA

Directly signing a server certificate with the root CA is acceptable for a local or isolated development environment. For production PKI, keep the root CA offline and sign server certificates through an intermediate CA instead.

This operation is performed in the CA environment, independently of the Camunda host and repository. Transfer `orchestration.csr` to the CA administrator. Never transfer the root CA private key to the Camunda host.

The default command copies the requested extensions, including SAN, Key Usage, and Extended Key Usage, from the CSR into the issued certificate. The CA administrator must inspect and approve these values before signing.

On the CA signing host, adjust these example paths and sign the CSR:

```bash
CA_CERT=/secure/ca/root-ca.crt
CA_KEY=/secure/ca/root-ca.key
CSR_FILE=/secure/ca/requests/orchestration.csr
ISSUED_CERT=/secure/ca/issued/orchestration-issued.crt

openssl x509 -req \
  -in "$CSR_FILE" \
  -CA "$CA_CERT" \
  -CAkey "$CA_KEY" \
  -CAcreateserial \
  -out "$ISSUED_CERT" \
  -days 825 -sha256 \
  -copy_extensions copy
```

If CA policy requires different SAN values or extensions, replace `-copy_extensions copy` with the CA-controlled signing template:

```bash
EXTENSIONS_FILE=/secure/ca/policy/openssl-signing-extensions.cnf

openssl x509 -req \
  -in "$CSR_FILE" \
  -CA "$CA_CERT" \
  -CAkey "$CA_KEY" \
  -CAcreateserial \
  -out "$ISSUED_CERT" \
  -days 825 -sha256 \
  -extfile "$EXTENSIONS_FILE" \
  -extensions server_cert
```

> **SAN precedence:** In the override command, the SAN values in `openssl-signing-extensions.cnf` are authoritative. If they differ from the CSR, the issued certificate contains the SAN values from the signing template, replacing the requested SAN values.

Verify that the resulting leaf certificate contains the expected subject, SAN, Key Usage, and Extended Key Usage:

```bash
openssl x509 -in "$ISSUED_CERT" \
  -noout -subject -issuer -dates -text
```

Return only the issued leaf certificate and the public CA chain to the Camunda operator. Do not return or expose the CA private key. When the leaf certificate is signed directly by the root CA, no intermediate certificate is required.

### Prepare the server certificate chain

Perform the remaining steps on the Camunda host after receiving the issued leaf certificate and public CA chain. Run the commands from the repository's `docker-compose/` directory and place the CA response in its `certificates/` directory. If the CA returns DER-encoded files, convert the leaf certificate to PEM first:

```bash
openssl x509 -inform DER \
  -in certificates/orchestration-issued.cer \
  -out certificates/orchestration-issued.crt
```

Build `tls.crt` with the leaf certificate first, followed by any intermediate CA certificates. Do not normally include the root CA in the server chain:

```bash
cp certificates/orchestration-issued.crt certificates/tls.crt
# Run this only when the CA provides one or more intermediate certificates.
cat certificates/intermediate-ca.crt >> certificates/tls.crt
chmod 644 certificates/tls.crt
```

Keep the root CA certificate separately as `certificates/root-ca.crt`. Create the trust file used by Connectors from this public root CA certificate:

```bash
cp certificates/root-ca.crt certificates/ca.crt
chmod 644 certificates/ca.crt
```

The server presents `tls.crt`, which contains the leaf certificate and any intermediate certificates. Connectors trusts `ca.crt`, which contains the root CA trust anchor. Do not append the root CA to the server chain.

### Verify the certificate

The certificate and private key fingerprints must match:

```bash
openssl x509 -in certificates/tls.crt -pubkey -noout | openssl sha256
openssl pkey -in certificates/tls.key -pubout | openssl sha256
```

Verify the chain and both DNS names used by clients:

```bash
openssl verify -CAfile certificates/root-ca.crt \
  -untrusted certificates/intermediate-ca.crt certificates/orchestration-issued.crt
openssl verify -CAfile certificates/root-ca.crt \
  -untrusted certificates/intermediate-ca.crt \
  -verify_hostname localhost certificates/orchestration-issued.crt
openssl verify -CAfile certificates/root-ca.crt \
  -untrusted certificates/intermediate-ca.crt \
  -verify_hostname orchestration certificates/orchestration-issued.crt
```

Omit `-untrusted certificates/intermediate-ca.crt` from all three commands when the CA issued the leaf certificate directly from the root CA.

After starting the stack, verify the live gRPC endpoint:

```bash
openssl s_client -connect localhost:26500 -servername localhost \
  -CAfile certificates/ca.crt -verify_hostname localhost -brief
```

The command must report `Verification: OK`. For Camunda Modeler or another host client, trust `ca.crt` in the operating system or application trust store. Never distribute or import `tls.key` on clients.
