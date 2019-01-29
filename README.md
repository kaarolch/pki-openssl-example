# pki-openssl-example
PKI base on openssl

1. Create OpenSSL folder structure, base path `pki-example/ca`.

```bash
mkdir -p pki-example/ca/{certs,crl,newcerts,private}
cd pki-example/ca/
chmod 700 private
touch index.txt
echo 01 > serial
```

Folder structure and short description:

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
