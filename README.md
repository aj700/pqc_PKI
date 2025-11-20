# Hybrid PKI Implementation (ECDSA + Dilithium)

This project demonstrates the implementation of a hybrid Public Key Infrastructure (PKI) that combines Elliptic Curve Digital Signature Algorithm (ECDSA) with a post-quantum cryptography (PQC) algorithm Dilithium. 
The setup includes a Root CA, Intermediate CA, and subscriber certificates, providing a complete chain of trust while ensuring quantum resistance.

- *Link to the docker environment setup is available in the references.*

## Overview

The hybrid approach combines P-384 (ECDSA) with Dilithium3, providing security against both classical and quantum computer attacks. This setup ensures backward compatibility( need openssl application to updated at your side) while preparing for the post-quantum era.

### About MLDSA (Dilithium)

Dilithium, is now standardized as MLDSA (Module-Lattice-Based Digital Signature Standard) in FIPS 204. It is NIST's chosen post-quantum digital signature algorithm at the moment. 
(NIST will soon standardize the FALCON algorithm which has smaller Key and Signature sizes and better suited for embedded systems as well.)

### Why Hybrid Algorithms?

1. **Quantum Threat**
   - Large-scale quantum computers could break current public-key cryptography (RSA, ECC) using Shor's algorithm
   - users need to prepare for "harvest now, decrypt later" attacks where encrypted data is stored until quantum computers become available
   - NIST estimates quantum computers capable of breaking current cryptography could exist within 10-15 years

2. **Transition Security**
   - Hybrid certificates maintain security even if one algorithm is compromised
   - If the classical algorithm is broken by quantum computers, the PQC algorithm maintains security
   - If a PQC algorithm is found to have weaknesses, the classical algorithm with higher key length maintains security
   - This approach provides confidence during the transition period

3. **Backward Compatibility**
   - Allows for staged migration of infrastructure without service disruption

4. **Future-Proofing**
   - Systems deployed today need to consider quantum threats within their operational lifespan

### Directory Structure
```
pqc_PKI/
├── PKIsubscriber/
│   ├── certs/
│   ├── csr/
│   └── private/
├── intermediateCA/
│   ├── certs/
│   ├── csr/
│   ├── newcerts/
│   └── private/
└── rootCA/
    ├── certs/
    ├── crl/
    ├── newcerts/
    ├── private/
    └── rootCA.conf
```

## Implementation Steps

### 1. Create Directory Structure
```bash
mkdir -p pqc_PKI/{rootCA,intermediateCA,PKIsubscriber}
cd pqc_PKI
mkdir -p rootCA/{certs,crl,newcerts,private}
mkdir -p intermediateCA/{certs,csr,newcerts,private}
mkdir -p PKIsubscriber/{certs,csr,private}
```

### 2. Generate Root CA

```bash
# Generate hybrid key pair
openssl genpkey -algorithm p384_dilithium3 -out rootCA/private/rootCA_p384_dilithium3.key

# Create configuration file rootCA.conf
# (Configuration content provided below)

# Generate and sign root certificate
openssl req -config rootCA/rootCA.conf -key rootCA/private/rootCA_p384_dilithium3.key \
    -new -x509 -days 7500 -extensions v3_ca -out rootCA/certs/rootCA_pqc.crt
```

### 3. Generate Intermediate CA

```bash
# Generate hybrid key pair
openssl genpkey -algorithm p384_dilithium3 \
    -out intermediateCA/private/intermediateCA_p384_dilithium3.key

# Create CSR
openssl req -config intermediateCA/interCA.conf \
    -key intermediateCA/private/intermediateCA_p384_dilithium3.key \
    -new -out intermediateCA/csr/interCA_pqc.csr

# Sign intermediate certificate
openssl ca -config rootCA/rootCA.conf -extensions v3_intermediate_ca \
    -days 3650 -notext -md sha384 \
    -in intermediateCA/csr/interCA_pqc.csr \
    -out intermediateCA/certs/interCA_pqc.crt
```

### 4. Generate Subscriber Certificate

```bash
# Generate hybrid key pair
openssl genpkey -algorithm p384_dilithium3 \
    -out PKIsubscriber/private/PKIsubscriber_p384_dilithium3.key

# Create CSR
openssl req -config subscriber.conf \
    -key PKIsubscriber/private/PKIsubscriber_p384_dilithium3.key \
    -new -out PKIsubscriber/csr/PKIsubscriber_pqc.csr

# Sign subscriber certificate
openssl ca -config intermediateCA/interCA.conf -extensions subscriber_cert \
    -days 365 -notext -md sha384 \
    -in PKIsubscriber/csr/PKIsubscriber_pqc.csr \
    -out PKIsubscriber/certs/PKIsubscriber_pqc.crt
```

## Configuration Files

### Root CA Configuration (rootCA.conf)
```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = /home/user/pqc_PKI/rootCA
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

private_key       = $dir/private/rootCA_p384_dilithium3.key
certificate       = $dir/certs/rootCA_pqc.crt

crlnumber         = $dir/crlnumber
crl               = $dir/crl/rootca.crl
crl_extensions    = crl_ext
default_crl_days  = 30

default_md        = sha384
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 7500
preserve          = no
policy            = policy_strict

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress           = optional

[ req ]
default_bits        = 3072
distinguished_name  = req_distinguished_name
string_mask        = utf8only
default_md         = sha384
x509_extensions    = v3_ca

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
stateOrProvinceName            = State or Province Name
localityName                   = Locality Name
0.organizationName             = Organization Name
organizationalUnitName         = Organizational Unit Name
commonName                     = Common Name
emailAddress                   = Email Address

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```

### Intermediate CA Configuration (interCA.conf)
```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = /home/user/pqc_PKI/intermediateCA
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

private_key       = $dir/private/intermediateCA_p384_dilithium3.key
certificate       = $dir/certs/interCA_pqc.crt

crlnumber         = $dir/crlnumber
crl               = $dir/crl/intermediate.crl
crl_extensions    = crl_ext
default_crl_days  = 30

default_md        = sha384
name_opt         = ca_default
cert_opt         = ca_default
default_days     = 375
preserve         = no
policy           = policy_loose

[ policy_loose ]
countryName             = match
stateOrProvinceName     = match
localityName            = optional
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress           = optional

[ req ]
default_bits        = 3072
distinguished_name  = req_distinguished_name
string_mask        = utf8only
default_md         = sha384

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
stateOrProvinceName            = State or Province Name
localityName                   = Locality Name
0.organizationName             = Organization Name
organizationalUnitName         = Organizational Unit Name
commonName                     = Common Name
emailAddress                   = Email Address

[ subscriber_cert ]
basicConstraints = CA:FALSE
subjectAltName = DNS:www.eu.idp.com, DNS:www.token-signing.eu.idp.com
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth, codeSigning
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
```

### Subscriber Configuration (subscriber.conf)
```ini
[ req ]
default_bits = 3072
prompt = no
default_md = sha384
distinguished_name = dn
x509_extensions = ext

[ dn ]
C = SE
ST = Gothenburg
L = Lundby
O = Volvo AB
OU = Volvo Technology
CN = www.exampledomain.com

[ ext ]
subjectAltName = DNS:www.exampledomain.com
keyUsage = critical, digitalSignature, keyEncipherment, codeSigning
extendedKeyUsage = serverAuth, clientAuth, codeSigning
```

## Create Subscriber certificate chain for distribution

```bash
# Create PKCS#7 bundle for distribution of subject certificate.
openssl crl2pkcs7 -nocrl \
    -certfile PKIsubscriber/certs/PKIsubscriber_pqc.crt \
    -certfile intermediateCA/certs/interCA_pqc.crt \
    -out Subscriber_cert_chain.p7b
```

## Certificate Chain Verification

```bash
# Verify the certificate chain
openssl verify -show_chain -CAfile rootCA/certs/rootCA_pqc.crt \
    -untrusted intermediateCA/certs/interCA_pqc.crt \
    PKIsubscriber/certs/PKIsubscriber_pqc.crt
```
![image](https://github.com/user-attachments/assets/f9db2815-dac9-4b95-a78e-9048bfee6a1a)

## Getting Started

This study uses the Open Quantum Safe (OQS) project's fork of OpenSSL. 
I have created a ready-to-use Docker container available with all prerequisites installed:
The container provides a complete example environment for experimenting with hybrid PKI using p384_dilithium3, which combines ECDSA P-384 with MLDSA (Dilithium3) in a hybrid scheme.

### how to use the container? 
```bash
# Pull the Docker image
docker pull arunjr/pqc_studies:latest

# Run the container
docker run -it arunjr/pqc_studies:latest

# You now have access to an environment with:
# - the default openssl build itself is from OQS project so you are good to go for PQC expirements.
```

## Security Considerations

1. **Key Storage**
   - private keys must be stored securely and CA keys shall be offline
   - Use appropriate access control permissions for all private key files

2. **Certificate Management**
   - Regular certificate validation, renewal and revocation checks are not considered in this scope.

## License

This project is open source and licensed under the MIT License, a copy is available in the same repo.
You are free to use it for commercial or private use.

## References

1. [Docker Container for this Project](https://hub.docker.com/repository/docker/arunjr/pqc_studies/general)
2. [Open Quantum Safe (OQS) Project](https://openquantumsafe.org/)
3. [NIST Post-Quantum Cryptography Standardization](https://csrc.nist.gov/projects/post-quantum-cryptography)
4.  Stack overflow… countless pages.

## Contributing

Please do share refinements, upgrades etc via pull requests.
