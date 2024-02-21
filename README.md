Table of Contents
=================

* [Instructions](#instructions)
    * [Get Gitea binary](#get-gitea-binary)
    * [Run gitea as standalone server](#run-gitea-as-standalone-server)
    * [Deploy gitea on a k8s cluster](#deploy-gitea-on-a-k8s-cluster)
    * [Run gitea as container](#run-gitea-as-container)
* [Test me](#test-me)
* [PKI](#pki)
  * [Gitea client](#gitea-client)
  * [OpenSSL](#openssl)
  * [mkcert](#mkcert)
  * [minica](#minica)

## Instructions

This project explains how to run locally (or using docker) a gitea instance like how to create a selfsigned CA certificate exposing 
the server using HTTPS with minimal effort ;-)

### Get Gitea binary

- Download the binary (see [doc](https://docs.gitea.com/installation/install-from-binary))
```bash
wget -O gitea https://dl.gitea.com/gitea/1.21.3/gitea-1.21.5-darwin-10.12-arm64
chmod +x gitea
```
or install it using `homebrew` tool: `brew install gitea`

Next, verify that the binary is working: `gitea -h`

### Run gitea as standalone server

When this is done, you will have to create a config app.ini file with the following content:

```bash
mkdir -p custom/conf
cat <<EOF > custom/conf/app.ini
[security]
INSTALL_LOCK = true

[database]
DB_TYPE = sqlite3
SSL_MODE = disable
PATH = ./data/gitea.db
LOG_SQL = false

[server]
PROTOCOL = http
HTTP_PORT = 3333
EOF
```
- Launch the gitea server
```bash
export GITEA_WORK_DIR=$HOME/code/gitea/my-gitea
export GITEA_CUSTOM=$GITEA_WORK_DIR/custom
gitea web
```
- Create an admin user
```bash
export GITEA_WORK_DIR=$HOME/code/gitea/my-gitea
export GITEA_CUSTOM=$GITEA_WORK_DIR/custom
gitea admin user create --admin --username gitea_admin --password gitea_admin --email "gitea@local.domain" --must-change-password=false
gitea admin user change-password --username gitea_admin --password gitea_admin
```
- Next, you can customize the `custom/conf/app.ini` file to by example configure the TLS section
```ini
[server]
PROTOCOL = https
CERT_FILE = ../pki/gitea/cert.pem
KEY_FILE = ../pki/gitea/key.pem
```
**NOTE**: Don't forget top restart the server

### Deploy gitea on a k8s cluster

Generate first a new selfsigned CA certificate and create the secret within `gitea` namespace
```bash
gitea cert --host gitea.localtest.me --ca
mv cert.pem gitea-localtest-me.pem
mv key.pem gitea-localtest-me.key
k -n gitea create secret tls gitea-tls --cert=gitea-localtest-me.pem --key=gitea-localtest-me.key
```
**Important**: Don't forget to set this env variable before to start backstage
```bash
export NODE_EXTRA_CA_CERTS=$HOME/code/gitea/my-gitea/pki/gitea/gitea-localtest-me.pem 
```

Generate the helm values and deploy the helm chart
```bash
cat <<EOF > helm-values.yml
redis-cluster:
  enabled: false
postgresql:
  enabled: false
postgresql-ha:
  enabled: false

persistence:
  enabled: false

gitea:
  admin:
    username: "gitea_admin"
    password: "gitea_admin"
    email: "gi@tea.com"
  config:
    database:
      DB_TYPE: sqlite3
    session:
      PROVIDER: memory
    cache:
      ADAPTER: memory
    queue:
      TYPE: level

service:
  ssh:
    type: NodePort
    nodePort: 32222
    externalTrafficPolicy: Local

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: gitea.localtest.me
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: gitea-tls
      hosts:
        - gitea.localtest.me
EOF
helm install gitea gitea-charts/gitea -n gitea --create-namespace -f helm-values.yml
```
To remove it
```bash
helm uninstall gitea -n gitea
```

### Run gitea as container

- Create a docker compose file as documented [here](https://docs.gitea.com/installation/install-with-docker)
```bash
cat << EOF > docker-compose.yml
version: "3"

networks:
  gitea:
    external: false

services:
  server:
    image: gitea/gitea:latest
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3333:3000"
      - "222:22"
EOF
```
and launch the container
 
```bash
docker-compose down; docker-compose up -d; docker-compose logs -f
```

## Test me

See: [curl-gitea.md](curl-gitea.md)

## PKI

This section describes different approaches to generate a selfsigned certificate for the `localhost` including also a [CA](https://en.wikipedia.org/wiki/Certificate_authority) (not trusted) certificate to be used by the gitea server.

To add the certificate generated to your keychain tool, execute by example on macOS the following command:
`sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain <CERTIFICATE_FILE>`

**NOTE**: You can also use [certbot](https://github.com/certbot/certbot) if you would like to generate a certificate for a domain name (e.g. gitea.example.com, etc.). 

**IMPORTANT**: If you plan to run gitea on kubernetes/openshift, then we recommend to use the [cert-manager](https://cert-manager.io/docs/) tool to generate a certificate for your domain name. See this project: https://github.com/snowdrop/pki?tab=readme-ov-file#create-a-pkcs12-using-cert-manager

#### Gitea client

To generate a certificate which is also its own CA & key
```bash
mkdir -p pki/gitea; cd pki/gitea
gitea cert --host localhost -ca
```
#### OpenSSL

1. Generate a private key and CA certificate:
```bash
mkdir openssl; cd openssl
openssl req -x509 -nodes -new -sha512 \
  -days 365 \
  -newkey rsa:4096 \
  -keyout ca.key \
  -out ca.crt \
  -subj "/C=BE/CN=Snowdrop-CA"
```
2. Check the certificate file
`openssl x509 -text -noout -in ca.crt`

3. Generate a x509 v3 extension file:
```bash
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names 
[alt_names]
# Local hosts
DNS.1 = localhost
IP.1 = 127.0.0.1
EOF
```
5. Generate a private key and certificate sign request (CSR) for localhost
```bash
openssl req -new -nodes -newkey rsa:4096 \
 -keyout localhost.key \
 -out localhost.csr \
 -subj "/C=BE/L=Silenrieux/O=Snowdrop/CN=localhost"
```

6. Generate a certificate signed using the CA cert using the CSR:
```bash
openssl x509 -req -sha512 -days 365 \
  -extfile v3.ext \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -in localhost.csr \
  -out localhost.crt
```
#### mkcert

https://github.com/FiloSottile/mkcert

```bash
mkcert localhost
ls -la
total 16
drwxr-xr-x@ 4 cmoullia  staff   128 Jan  3 17:16 .
drwxr-xr-x@ 6 cmoullia  staff   192 Jan  3 17:16 ..
-rw-------@ 1 cmoullia  staff  1704 Jan  3 17:16 localhost-key.pem
-rw-r--r--@ 1 cmoullia  staff  1517 Jan  3 17:16 localhost.pem

openssl s_server -accept 8443 -key localhost-key.pem -cert localhost.pem -www
```
#### minica

https://github.com/jsha/minica/issues/66

```bash
brew install minica
minica -domains localhost -ip-addresses 127.0.0.1 -ca-cert ca.crt -ca-key ca.key

openssl x509 -noout -text -in ca.crt
openssl x509 -noout -text -in localhost/cert.pem
```