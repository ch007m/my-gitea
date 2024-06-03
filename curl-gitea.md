# Curl commands

Before to  curl a gitea instance, set the following properties and get a token

```bash
export GITEA_API_URL="https://localhost:3333/api"
export GITEA_USERNAME="gitea_admin"
export GITEA_PASSWORD="gitea_admin"

TOKEN=$(curl -s -k -H "Content-Type: application/json" -d '{"name":"init","scopes": ["write:user", "write:admin", "write:repository"]}' -u $GITEA_USERNAME:$GITEA_PASSWORD $GITEA_API_URL/v1/users/$GITEA_USERNAME/tokens | jq -r .sha1)
echo $TOKEN
```

## Create a user

```bash
curl -k -X 'POST' \
  "$GITEA_API_URL/v1/admin/users" \
  -H 'accept: application/json' \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD" \
  -H 'Content-Type: application/json' \
  -d '{
  "email": "user1@cnoe.io",
  "full_name": "user1",
  "login_name": "user1",
  "must_change_password": false,
  "password": "user11234",
  "restricted": false,
  "visibility": "public",
  "send_notify": false,
  "username": "user1"
}'
```

## Import key

```bash
curl -k -X POST\
  "$GITEA_API_URL/v1/user/keys" \
  -H 'accept: application/json' \
  -u "gitea_admin:gitea_admin" \
  -H 'Content-Type: application/json' \
  -d '{
   "id": 1, // This is the ID of the gitea_admin account
   "key": "ssh-rsa AAAAB3Nz...KBE1ecj7WSL9",
   "read_only": true,
   "title": "org1@gmail.com"
}'
```

## Create a new org

```bash
export ORG_NAME="org1"
curl -kv -X POST \
  "$GITEA_API_URL/v1/orgs" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD" \
  -d '{"username": "'"$ORG_NAME"'"}'
```

## Get a new org

```bash
export ORG_NAME="org1"
curl -k -X GET \
  "$GITEA_API_URL/v1/orgs/$ORG_NAME" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD"
```

## Create a repository in an org

```bash
export REPO_NAME="dummy"
export REPO_DESCRIPTION="dummy dummy"
export ORG_NAME="org1"

curl -kv \
  "$GITEA_API_URL/v1/orgs/$ORG_NAME/repos" \
  -H 'accept: application/json' \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD" \
  -H 'Content-Type: application/json' \
  -d '{
     "auto_init": true,
     "default_branch": "main",
     "description": "'"$REPO_DESCRIPTION"'",
     "name": "'"$REPO_NAME"'",
     "readme": "Default",
     "private": true
}'
```

## Get an org's repo

```bash
export ORG_NAME="org1"
export REPO_NAME="dummy"
curl -kv \
  "$GITEA_API_URL/v1/repos/$ORG_NAME/$REPO_NAME" \
  -H 'accept: application/json'
```

## Delete an org's repo

```bash
export REPO_NAME="dummy"
export ORG_NAME="org1"
curl -k -X 'DELETE' \
  "$GITEA_API_URL/v1/repos/$ORG_NAME/$REPO_NAME" \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json'
```

## Get repo content of file: catalog-info.yaml

```bash
export REPO_NAME="test1"
export ORG_NAME="org1"
curl -k -X GET \
  "$GITEA_API_URL/v1/repos/$ORG_NAME/$REPO_NAME/contents/catalog-info.yaml?ref=main" \
  -H 'accept: application/json' \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD"
```

## Create a user's repository

```bash
export REPO_NAME="dummy"
export REPO_DESCRIPTION="dummy"
curl -v \
  "$GITEA_API_URL/v1/user/repos" \
  -H 'accept: application/json' \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD" \
  -H 'Content-Type: application/json' \
  -d '{ "default_branch": "main",
  "description": "dummy",
  "name": "dummy",
  "private": false,
  "readme": "dummy" }'
```

## Delete a user's repo

```bash
curl -X 'DELETE' \
  "$GITEA_API_URL/v1/repos/$GITEA_USERNAME/$REPO_NAME" \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json'
```

## Push a file to a user's repo

```bash
curl -X 'POST' \
  "$GITEA_API_URL/v1/repos/$GITEA_USERNAME/dummy/contents/README.md" \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "author": {
    "email": "user1@snowdrop.dev",
    "name": "user1"
  },
  "branch": "main",
  "committer": {
    "email": "user1@snowdrop.dev",
    "name": "user1"
  },
  "content": "IyMgRHVtbXkgcHJvamVjdAoKVGhpcyBpcyBhIGR1bW15IHByb2plY3QgdG8gdGVzdCB0aGUgZ2l0ZWEgc2VydmVyLg==",
  "dates": {
    "author": "2023-12-14T16:51:11.906Z",
    "committer": "2023-12-14T16:51:11.906Z"
  },
  "message": "initial upload",
  "signoff": true
}'
```

## Commit a message to a user's repo

```bash
curl -X 'POST' \
  "$GITEA_API_URL/v1/repos/user1/dummy/statuses/123456" \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "context": "123456",
  "description": "123456",
  "state": "string",
  "target_url": "string"
}'
```
