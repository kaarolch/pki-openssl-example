# pki-openssl-example
PKI base on OpenSSL
## Required packages
Fedora/Centos:
*  openssl
*  nginx

Use `yum` to install them:
```bash
sudo yum install openssl nginx
```
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

Certificate Authority folder structure and short description:

```bash
|
|- certs        #
|- crl          # folder for crl list
|- csr          # folder for certificate request
|- index.txt    # simple certificates database
|- openssl.conf # config file for OpenSSL
|- private      # here rsa keys generated via openssl are stored
|- serial       # store serial number
```
## Prepare OpenSSL configuration
The [openssl.conf](openssl.conf) includes all important OpenSSL settings to run root CA and also intermediate CA. The `OCSP` configuration is not included but could be found in [OpenSSL doc](https://www.openssl.org/docs/).
 This configuration include three certificate profiles:
 - ca profile - section `[v3_ca]`
 - server profile - section `[srv_crt]`
 - user profile - section  `[usr_crt]`

## Generate root self-signed certificate

1.  Generate RSA key ( You need to provide pass phrase, because `-aes256 ` is used ):
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
    A new certificate has `CA extension` enabled.
4.  Convert CA cert to DER format:
    ```bash
    openssl x509 -outform der -in certs/ca.pem -out certs/ca.der
    ```
## Generate server certificate and add it to eample web server

Commands from this section generate server key, csr and certificate on CA side. Steps 1-3 could be also run on external server. After that the new CSR has to be transferred to CA server.

**Used domain name:** `pki-example.local`

### Generate server certificate

By default [openssl.conf](openssl.conf#L51) provide 2048 bit key. If you would like to generate 4096 key just add `4096` in the end of next command.

1.  Generate server rsa key:
    ```bash
    openssl genrsa -out private/pki-example.local-key.pem
    ```

2.  Use private rsa key to generate certificate request. It will be placed in [pki-example/ca/csr](pki-example/ca/csr):

    ```bash
    openssl req -config openssl.conf -key private/pki-example.local-key.pem  -new -sha256 -out csr/pki-example.local-csr.pem -subj "/C=PL/ST=LesserPoland/O=Example PKI/CN=pki-example.local"
    ```

3.  Below command displays CSR content:

    ```bash
    openssl req -in csr/pki-example.local-csr.pem -noout -text
    ```
4.  A CSR allows to generate a server certificate (you have to provide CA pass phase to encrypt ca key):

    ```bash
    openssl ca -config openssl.conf -extensions srv_cert -notext -md sha256 -in csr/pki-example.local-csr.pem -out certs/pkisudo cp private/pki-example.local-key.pem /etc/pki/nginx/private && sudo cat certs/pki-example.local-cert.pem certs/ca.pem | sudo tee -a /etc/pki/nginx/pki-example.local-bundle.pem &&
    sudo chown -R nginx:nginx /etc/pki/nginx &&
    sudo chmod 640 -R /etc/pki/nginx/private-example.local-cert.pem
    ```
5.  What was changed?:
    * index.txt has a new entry:
      ```bash
      V	200202145727Z		00	unknown	/C=PL/ST=LesserPoland/O=Example PKI/CN=pki-example.local
      ```
    * A serial was bumped to `01`
    * server cert was signed: [certs/pki-example.local-cert.pem](certs/pki-example.local-cert.pem) below command would display cert content:
    ```bash
    openssl x509 -in certs/pki-example.local-cert.pem -noout -text
    ```
6.  New cert could be verified via command:
    ```bash
    openssl verify -CAfile certs/ca.pem certs/pki-example.local-cert.pem
    ```
7.  Now server cert [certs/pki-example.local-cert.pem](certs/pki-example.local-cert.pem), server key [private/pki-example.local-key.pem](private/pki-example.local-key.pem) and ca certificate [certs/ca.pem](certs/ca.pem) could be transfer to appropriate server.
8.  Generated cert could be converted to pkcs12 via command:
    ```bash
    openssl pkcs12 -export -out certs/pki-example.local-cert.pfx -inkey private/pki-example.local-key.pem -in certs/pki-example.local-cert.pem -certfile certs/ca.pem
    ```
    Above command is required a pass phase for pkcs12.  

### Use new certificate on NGINX server

1.  Install nginx on new server (CentOS/ Fedora):
    ```bash
    sudo yum install nginx
    ```
2.  Prepare NGINX folder structure:
    ```bash
    mkdir -p /etc/pki/nginx/private
    chown -R nginx:nginx /etc/pki/nginx
    ```
3.  Copy server key and create cert bundle:
    ```bash
    sudo cp private/pki-example.local-key.pem /etc/pki/nginx/private && sudo cat certs/pki-example.local-cert.pem certs/ca.pem | sudo tee -a /etc/pki/nginx/pki-example.local-bundle.pem &&
    sudo chown -R nginx:nginx /etc/pki/nginx &&
    sudo chmod 640 -R /etc/pki/nginx/privatehttps://github.com/kaarolch/pki-openssl-example
    ```
4.  Configure nginx to use new certificate. In [/etc/nginx/nginx.conf](/etc/nginx/nginx.conf) uncomment tsl section and correct `ssl_certificate` and `ssl_certificate_key`
    ```bash
    server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  _;https://github.com/kaarolch/pki-openssl-example
        root         /usr/share/nginx/html;

        ssl_certificate "/etc/pki/nginx/pki-example.local-https://github.com/kaarolch/pki-openssl-examplebundle.pem";
        ssl_certificate_key "/etc/pki/nginx/private/pki-example.local-key.pem";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers PROFILE=SYSTEM;
        ssl_prefer_server_ciphers on;
        include /etc/nginx/default.d/*.conf;
        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
    ```
5.  To test nginx locally add `pki-example.local` to `/etc/hosts`:
    ```bash
    127.0.0.1 pki-example.local
    ```
