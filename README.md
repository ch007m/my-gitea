## Instructions

### Gitea binary

- Download the binary
```bash
wget -O gitea https://dl.gitea.com/gitea/1.21.3/gitea-1.21.3-darwin-10.12-arm64
chmod +x gitea

or 

brew install gitea

gitea -h
```
- Create app.ini file
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
- Launch gitea server
```bash
GITEA_WORK_DIR=$HOME/code/gitea/my-gitea \
GITEA_CUSTOM=$GITEA_WORK_DIR/custom \
  gitea web -c custom/conf/app.ini
```

- Create an admin user
```bash
export GITEA_WORK_DIR=$HOME/code/gitea/my-gitea
export GITEA_CUSTOM=$GITEA_WORK_DIR/custom
gitea admin user create --admin --username gitea_admin --password gitea_admin --email "gitea@local.domain" --must-change-password=false
gitea admin user change-password --username gitea_admin --password gitea_admin
```
- Next, you can cutomize the `custom/conf/app.ini` file and restart the server
```ini
[server]
PROTOCOL = https
CERT_FILE = ../pki/gitea/cert.pem
KEY_FILE = ../pki/gitea/key.pem
```

### Docker compose

- Create a docker compose file 
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

### PKI

#### Gitea client

To generate a certificate which is also its own CA & key
```bash
mkdir -p pki; cd pki
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