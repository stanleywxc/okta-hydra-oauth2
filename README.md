# okta-hydra-oauth2

#### Initial Setup Steps

```
docker network create hydra-net
```

The following setup will use my existing postgres database, which is in postgres-db network If you don't have postgres database setup, you can create on with the command below and use hydra-net as its network. I will use mine so I will join the 'hydra-net'

```
Skip this step, if you have your own postgres database

docker run --network hydra-net \
  --name hydra-postgres \
  -e POSTGRES_USER=hydra \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=hydra \
  -d postgres:9.6
```

```
This secret is a critical one, you should save it to somewhere since it is used for encrypt data at rest.

export SECRETS_SYSTEM=$(export LC_CTYPE=C; cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1)

```

```
Obviously, you need to create a user 'hydra' and database 'hydra' first

export DSN=postgres://hydra:secret@postgres-db:5432/hydra?sslmode=disable

```


```
docker run -it --rm \
  --network postgres-net \
  oryd/hydra:v1.7.4 \
  migrate sql --yes $DSN

docker run -d \
  --name hydra-hydra \
  --network postgres-net \
  -p 9000:4444 \
  -p 9001:4445 \
  -e SECRETS_SYSTEM=$SECRETS_SYSTEM \
  -e DSN=$DSN \
  -e URLS_SELF_ISSUER=http://127.0.0.1:9000/ \
  -e URLS_CONSENT=http://127.0.0.1:9020/consent \
  -e URLS_LOGIN=http://127.0.0.1:9020/login \
  oryd/hydra:v1.7.4 serve all --dangerous-force-http

# Attach the container to 'hydra-net' network
docker network connect hydra-net hydra-hydra


Check if it is running:

docker run --rm -it oryd/hydra:v1.7.4 help


docker run --rm -it \
  --network hydra-net \
  oryd/hydra:v1.7.4 \
  clients create \
    --endpoint http://hydra-hydra:4445 \
    --id test-user \
    --secret test-password \
    --grant-types client_credentials \
    --response-types token,code


docker run --rm -it \
  --network hydra-net \
  oryd/hydra:v1.7.4 \
  token client \
    --client-id test-user \
    --client-secret test-password \
    --endpoint http://hydra-hydra:4444


docker run --rm -it \
  --network hydra-net \
  oryd/hydra:v1.7.4 \
  token introspect \
    --endpoint http://hydra-hydra:4445 \
    xUOaZzW5E_GTiWzVp-CGdYdi1zuGljMPClLLOoA9cKY.CG-Jei8PU3F5QyNwrwPx98yWcLFEG0fJgJqc30WdulU


docker run -d \
  --name ory-hydraexample--consent \
  -p 9020:3000 \
  --network hydra-net \
  -e HYDRA_ADMIN_URL=http://hydra-hydra:4445 \
  -e NODE_TLS_REJECT_UNAUTHORIZED=0 \
  oryd/hydra-login-consent-node:v1.7.4


docker run --rm -it \
  --network hydra-net \
  oryd/hydra:v1.7.4 \
  clients create \
    --endpoint http://hydra-hydra:4445 \
    --id test12@test.com \
    --secret password \
    -g authorization_code,refresh_token \
    -r token,code,id_token \
    --scope openid,offline \
    --callbacks http://127.0.0.1:9010/callback


docker run --rm -it \
  --network hydra-net \
  -p 9010:9010 \
  oryd/hydra:v1.7.4 \
  token user \
    --port 9010 \
    --auth-url http://127.0.0.1:9000/oauth2/auth \
    --token-url http://hydra-hydra:4444/oauth2/token \
    --client-id test12@test.com \
    --client-secret password \
    --scope openid,offline \
    --redirect http://127.0.0.1:9010/callback



```
