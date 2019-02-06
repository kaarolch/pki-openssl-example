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
echo 00 > crlnumber
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
    openssl ca -config openssl.conf -extensions srv_cert -notext -md sha256 -in csr/pki-example.local-csr.pem -out certs/pki-example.local-cert.pem &&
    sudo cp private/pki-example.local-key.pem /etc/pki/nginx/private &&
    sudo cat certs/pki-example.local-cert.pem certs/ca.pem | sudo tee -a /etc/pki/nginx/pki-example.local-bundle.pem &&
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
    sudo chmod 640 -R /etc/pki/nginx/private
    ```
4.  Configure nginx to use new certificate. In [/etc/nginx/nginx.conf](/etc/nginx/nginx.conf) uncomment tsl section and correct `ssl_certificate` and `ssl_certificate_key`
    ```bash
    server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        ssl_certificate "/etc/pki/nginx/pki-example.local-bundle.pem";
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
# Configure Certificate revocation list (CRL)

A CRL provides a list of certificates that have been revoked. Some serpki-example.localvices like OpenVPN server can use it to deny access.    

When a certificate authority signs a certificate, it could add the CRL location into the certificate.
In the certificate profile operator has to add section `crlDistributionPoints = URI:http://pki-example.local/crl.pem`.

> After the certificate was signed provided url can not be changed.

1.  Add `crlDistributionPoints` to openssl profile i.e. `usr_cert`:
    ```bash
    [ usr_cert ]pki-example.local-
    # Extensions for client certificates
    basicConstraints = CA:FALSE
    nsCertType = client, email
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid,issuer
    keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
    extendedKeyUsage = echo 00 > crlnumberclientAuth, emailProtection
    crlDistributionPoints = URI:http://pki-example.local/crl.pem
    ```
2.  Generate CRL list, the CRL list has to be regenerated each time certificate would be revoke:
    ```bash
    openssl ca -config openssl.cnf -gencrl -out crl/ca-crl.pem
    ```

3.  Generate client certificate for test CRL:
    ```bash
    openssl genrsa -out private/test-crl-key.pem &&
    openssl req -config openssl.conf -key private/test-crl-key.pem  -new -sha256 -out csr/test-crl-csr.pem -subj "/C=PL/ST=LesserPoland/O=Example PKI/CN=test-crl" &&
    openssl ca -config openssl.conf -extensions usr_cert -notext -md sha256 -in csr/test-crl-csr.pem -out certs/test-crl-cert.pem
    ```
4.  Revoke generated certificate:
    ```bash
    openssl ca -config openssl.conf -revoke certs/test-crl-cert.pem
    ```
    Result, second certificate in `index.txt` was revoked (R):
    ```bash
    V	200202165027Z		03	unknown	/C=PL/ST=LesserPoland/O=Example PKI/CN=pki-example.local
    R	200206214941Z	190206215134Z	04	unknown	/C=PL/ST=LesserPoland/O=Example PKI/CN=test-crl

    ```
5.  Refresh CRL list:
    ```bash
    openssl ca -config openssl.conf -gencrl -out crl/ca-crl.pem
    ```
6.  Check CRL content:
    ```bash
    openssl crl -in crl/ca-crl.pem -noout -text
    ```
    Result:
    ```
    Certificate Revocation List (CRL):
        Version 2 (0x1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = PL, ST = LesserPoland, O = Example PKI, CN = Root-CA
        Last Update: Feb  6 21:56:22 2019 GMT
        Next Update: Mar  8 21:56:22 2019 GMT
        CRL extensions:
            X509v3 Authority Key Identifier:
                keyid:59:E6:DD:84:E9:90:50:95:21:1A:F8:E2:DB:D3:64:50:FE:0C:26:CB

            X509v3 CRL Number:
                0
        Revoked Certificates:
            Serial Number: 04
                Revocation Date: Feb  6 21:51:34 2019 GMT
            Signature Algorithm: sha256WithRSAEncryption
                 45:76:87:12:4d:ff:dc:9d:e7:52:9d:3d:20:da:de:60:cb:08:
                 a4:da:b1:29:99:ef:cb:49:df:86:ab:95:3c:4a:94:b9:19:1a:
                 3e:18:47:44:bf:55:26:37:67:50:24:1c:88:96:4e:f8:9c:96:
                 d7:89:9a:d4:83:ed:db:79:52:4f:b3:a1:d1:96:bc:a8:32:45:
                 15:3e:e1:90:13:5a:df:17:c8:9b:55:3d:a5:15:ef:f6:75:98:
                 7d:a6:05:0b:d0:02:36:e0:98:17:9f:50:a6:60:b7:4c:e1:9e:
                 b8:e5:25:6f:39:f7:27:67:ab:a3:43:f8:d4:8d:f0:c3:05:81:
                 d7:fe:fc:1a:86:6b:a4:53:86:d2:3c:4d:73:f6:c1:f6:9c:51:
                 7d:b9:bb:03:be:19:fd:77:26:25:75:fc:8a:7e:d2:f3:07:a1:
                 28:5a:05:67:fa:70:3a:d5:98:cc:fd:98:a3:98:ed:52:36:5a:
                 de:61:03:a4:b0:d4:7b:91:96:7d:e4:11:0a:e8:29:2d:87:f7:
                 88:48:af:a5:0a:c1:53:24:52:7d:11:87:c6:52:8a:ae:12:97:
                 7d:26:70:d9:cf:69:7f:78:55:59:c0:e9:75:b3:cc:3a:c1:e3:
                 d8:78:88:1a:88:17:24:35:49:00:dc:46:47:b2:6a:bc:7a:bc:
                 63:f1:a8:dc:79:9c:89:1a:f4:b6:b4:18:e1:87:d3:6a:75:bf:
                 ee:44:18:0f:c9:8a:c8:39:58:65:b0:76:c3:9e:49:f2:c1:74:
                 22:fb:b2:a2:7f:2c:86:9d:e8:a7:f6:b9:1a:80:cf:37:de:25:
                 34:7a:96:aa:95:d3:88:75:c6:06:f2:49:fe:97:56:92:68:51:
                 81:7f:9b:9a:63:ce:fa:3c:27:0f:d4:e6:0a:1c:c0:ee:0f:d2:
                 a1:b6:e4:0a:8a:b2:74:36:39:d7:78:f1:87:ae:69:d4:a8:e2:
                 fc:aa:2a:d6:06:3a:8c:a2:9a:15:12:0b:1c:11:14:5d:5e:82:
                 ca:dc:63:70:d3:89:61:84:8b:11:23:9b:91:fc:2a:c9:1c:a8:
                 e9:41:d7:88:b3:bd:7b:92:64:35:c8:1a:77:32:71:56:f4:41:
                 6a:eb:98:ce:b9:f0:df:1c:45:91:3e:96:b4:4e:8b:5f:c6:aa:
                 38:0e:fc:7d:c2:7f:c5:2d:f3:84:3f:c3:29:ac:90:e0:56:ea:
                 7c:8b:a3:61:c4:fb:12:1a:42:17:a6:b7:3f:b0:1a:08:86:a2:
                 28:0e:75:11:85:9b:c4:ed:5d:72:bb:19:06:9a:18:ea:92:b9:
                 41:0a:33:23:c3:77:bf:80:29:ad:e4:16:77:17:5b:82:f7:84:
                 58:5f:a9:88:08:f8:d3:2a
    ```
7.  Enable nginx cert verification:
    ```bash
    sudo cp ca/certs/ca.pem /etc/pki/nginx/ &&
    sudo cp crl/ca-crl.pem /etc/pki/nginx &&
    sudo chown -R nginx:nginx /etc/pki/nginx
    ```
    update `nginx.conf` ssl section:
    ```bash
        ssl_client_certificate "/etc/pki/nginx/ca.pem";
        ssl_crl "/etc/pki/nginx/ca-crl.pem";
        ssl_verify_client on;  
    ```
    restart nginx:
    ```bash
    systemctl restart nginx
    ```
