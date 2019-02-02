# pki-openssl-example
PKI base on openssl

## Create OpenSSL folder structure
Base path:  `pki-example/ca`

```bash
mkdir -p pki-example/ca/{certs,crl,newcerts,private}
cp openssl.conf pki-example/ca/
sed -i 's?CHANGE_THIS_PATH?'`pwd`'?' ./pki-example/ca/openssl.conf
cd pki-example/ca/
chmod 700 private
touch index.txt
echo 01 > serial
```

CA folder structure and short description:

```bash
|
|- certs        #
|- crl          # folder for crl list
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
