# pki-openssl-example
PKI base on openssl

## Create OpenSSL folder structure
Base path:  `pki-example/ca`

```bash
mkdir -p pki-example/ca/{certs,crl,csr,newcerts,private}
cp openssl.conf pki-example/ca/
sed -i 's?CHANGE_THIS_PATH?'`pwd`'?' ./pki-example/ca/openssl.conf
cd pki-example/ca/
chmod 700 private
touch index.txt
echo 00 > serial
```

CA folder structure and short description:

```bash
|
|- certs        #
|- crl          # folder for crl list
|- csr           # folder for certificate request
|- index.txt    # simple certificates database
|- newcerts     #
|- openssl.conf # config file for OpenSSL
|- private      #
|- serial       # store serial number generated for a certificate
```
## Prepare OpenSSL configuration
The [openssl.conf](openssl.conf) includes all important OpenSSL settings to run root CA and also intermediate CA. The `OCSP` configuration is not included but could be found in [OpenSSL doc](https://www.openssl.org/docs/).
 This config include three certificate profiles:
 - CA profile - section `[v3_ca]`
 - server cert profile - section `[srv_crt]`
 - user cert profile - section  `[usr_crt]`

## Generate root self-signed certificate

1.  Generate RSA keys ( You need to provide pass phrase ):
    ```bash
    openssl genrsa -aes256 -out private/ca-key.pem 4096
    chmod 400 private/ca-key.pem
    ```
2.  Generate certificate (profile v3_ca, valid: 10 years) - *key pass phrase is required*:
    ```bash
    openssl req -config openssl.conf -key private/ca-key.pem -new -x509 -days 3650 -sha256 -extensions v3_ca -out certs/ca.pem
    chmod 444 certs/ca.pem
    ```
3.  By default each certificate is save as base64 encrypted text (PEM format). To verify it use below command:
    ```bash
     openssl x509 -noout -text -in certs/ca.pem
    ```
    Content:
    ```bash
    Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            0d:15:33:42:d8:0c:22:ab:ef:06:00:1b:ab:a0:a1:0f:7c:ea:71:26
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = PL, ST = LesserPoland, O = Example PKI, CN = Root-CA
        Validity
            Not Before: Feb  2 06:16:58 2019 GMT
            Not After : Dec 11 06:16:58 2028 GMT
        Subject: C = PL, ST = LesserPoland, O = Example PKI, CN = Root-CA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (4096 bit)
        ...
        X509v3 Basic Constraints: critical
               CA:TRUE
        ...
    ```
    New certificate has `CA extension` enabled.
4.  Convert CA cert to DER format:
    ```bash
    openssl x509 -outform der -in certs/ca.pem -out certs/ca.der
    ```
## Generate server certificate and add it to web server

By default [openssl.conf](openssl.conf) provide 2048 bit key during cert request. If you would like to generate 4096 key just add `4096` in the end of next command.

1.  Generate server rsa key:
    ```bash
    openssl genrsa -out private/pki-example.local-key.pem
    ```

2.  Use private key to generate certificate request. It would be placed in [pki-example/ca/csr](pki-example/ca/csr):

    ```bash
    openssl req -config openssl.conf -key private/pki-example.local-key.pem  -new -sha256 -out csr/pki-example.local-csr.pem -subj "/C=PL/ST=LesserPoland/O=Example PKI/CN=pki-example.local"
    ```

3.  Below command displays CSR content:

    ```bash
    openssl req -in csr/pki-example.local-csr.pem -noout -text
    ```
4.  CSR allows to generate server certificate (you have to provide CA pass phase to encrypt ca key):

    ```bash
    openssl ca -config openssl.conf -extensions srv_cert -notext -md sha256 -in csr/pki-example.local-csr.pem -out certs/pki-example.local-cert.pem
    ```
5.  What was changed?:
    * index.txt - has new entry:
      ```bash
      V	200202145727Z		00	unknown	/C=PL/ST=LesserPoland/O=Example PKI/CN=pki-example.local
      ```
    * serial was bumped to `01`
    * server cert was signed: [ca/certs/pki-example.local-cert.pem](ca/certs/pki-example.local-cert.pem) below command would display cert content:
    ```bash
    openssl x509 -in certs/pki-example.local-cert.pem -noout -text
    ```
6.  New cert could be verified via command:
    ```bash
    openssl verify -CAfile certs/ca.pem certs/pki-example.local-cert.pem
    ```
