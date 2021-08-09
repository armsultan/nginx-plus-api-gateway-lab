# Revoking JWTs

## Introduction

From time to time, it may be necessary to revoke or re‑issue an API client's
JWT. Using simple `map` and `if` blocks, we can deny access to an API client by
marking its JWT as revoked until the JWT's exp claim (expiration date) is
reached, at which point the map entry for that JWT can be safely removed.

```nginx
# Map - Check "sub" (Subject) Claim for revoked subject values
map $jwt_claim_sub $jwt_status {
    "bobby@example.com" "revoked";
    default  "";
}

server {

  #...

  # Demostrate revoking JWT
  location /products/ {

      if ( $jwt_status = "revoked" ) {
          return 403;
      }

      proxy_pass http://echo_http;

      # JWT in "Authorization" header as a Bearer Token (Default)
      # Using HS256 cryptographic algorithm
      auth_jwt "My secure site";
      auth_jwt_key_file jwk/hs256/secrets.jwk;

  }
}
```

In the NGINX Config above NGINX will do the following:

 * The Nginx configuration above checks the JWT "`sub`" (Subject) Claim value in
   a `map` and sets the variable, `$jwt_status` to "`revoked`" if the
   `$jwt_claim_sub` is a listed (denylisted) subject value.
 * In `location /products/`, the `if` block will return a `HTTP 403 forbidden`
   response when `$jwt_status = "revoked"`

For more details read:

 * [Authenticating API Clients with JWT and NGINX
   Plus](https://www.nginx.com/blog/authenticating-api-clients-jwt-nginx-plus/)
 * Nginx Module:
   [`ngx_http_auth_jwt_module`](http://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html)
 * JWT log format: [`/log_format/log_jwt.conf)`](/log_format/log_jwt.conf)

## Learning Objectives 

By the end of the lab you will be able to: 
 * Revoking Clients mapping on their JWT

## Inspect Nested NGINX configs saving JWT Claims and Array Data as varibles

1.  Enable the config `warehouse_api_simple.conf`

```bash
for x in /etc/nginx/api_conf.d/*.conf; do mv "$x" "${x%.conf}.conf_"; done
mv /etc/nginx/api_conf.d/warehouse_api_jwt.conf_ /etc/nginx/api_conf.d/warehouse_api_jwt.conf
ls /etc/nginx/api_conf.d
nginx -t && nginx -s reload
```

2. To reproduce these demos, we will use prepared JWTs in text files and insert
   them in our `curl` commands. You can find the nested JWT files in the
   following directory: `jwt/hs256/nested_jwt`

```
$ ls /etc/nginx/jwt/hs256/nested_jwt

expired_token_nested.jwt  invalid_token_nested.jwt  valid_token_nested_1.jwt  valid_token_nested_2.jwt
```

1. There are two valid tokens we can test with: `valid_token_nested_1.jwt` and
   `valid_token_nested_2.jwt`, each JWT has a unqiue `dept` value, as defined in
   the nested object `attributes > dept`. Lets inspect
   `valid_token_nested_1.jwt`:


```json
$ cd /etc/nginx/jwt/hs256/nested_jwt
$ cat valid_token_nested_1.jwt

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjI2ODkyNDg2NTEsInN1YiI6InhhbXBsZUBleGFtcGxlLmNvbSIsImF1ZCI6Im5naW54IiwiYXR0cmlidXRlcyI6eyJuYW1lIjoiWGF2aWVyIEFtcGxlIiwicm9vbSI6IkExMjMiLCJkZXB0IjoiRGVtb25zdHJhdGlvbnMifSwiZ3JvdXBzIjpbIkFkbWluaXN0cmF0b3IiLCJGb29iYXIiLCJCYXpwdWIiXX0.weZPtC8szT-ZHZnrz2SWLorV9Mio_KjZVt33v5flBzM

$ JWT=valid_token_nested_1.jwt
$ jq -R 'split(".") | .[0],.[1] | @base64d | fromjson' <<< $(cat "${JWT}")

{
  "alg": "HS256",
  "typ": "JWT"
}
{
  "exp": 2689248651,
  "sub": "xample@example.com",
  "aud": "nginx",
  "attributes": {
    "name": "Xavier Ample",
    "room": "A123",
    "dept": "Demonstrations"
  },
  "groups": [
    "Administrator",
    "Foobar",
    "Bazpub"
  ]
}
```

1. Lets also inspect `valid_token_nested_2.jwt`:


```json
$ cat valid_token_nested_2.jwt

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjI2ODkyNDg2NTEsInN1YiI6ImJvYmJ5QGV4YW1wbGUuY29tIiwiYXVkIjoibmdpbngiLCJhdHRyaWJ1dGVzIjp7Im5hbWUiOiJib2JieSBkaWdpdGFsIiwicm9vbSI6IkE0NTYiLCJkZXB0IjoiT3BlcmF0aW9ucyJ9LCJncm91cHMiOlsiSW50ZXJuIiwiRm9vYmFyIiwiQmF6cHViIl19.RtyIeFRHZkMZKq_qMBQlJqWqXRPhoOU5fR8WcNj3Yn0

$ JWT=valid_token_nested_2.jwt
$ jq -R 'split(".") | .[0],.[1] | @base64d | fromjson' <<< $(cat "${JWT}")

{
  "alg": "HS256",
  "typ": "JWT"
}
{
  "exp": 2689248651,
  "sub": "bobby@example.com",
  "aud": "nginx",
  "attributes": {
    "name": "bobby digital",
    "room": "A456",
    "dept": "Operations"
  },
  "groups": [
    "Intern",
    "Foobar",
    "Bazpub"
  ]
}

```

### Revoking JWTs

In this example, we will validate JWT claims by checking the JWT sub (Subject)
claim value has not been revoked. This example only the sub claim
`bobby@example.com` is revoked

1. Move into folder with the nested JWT claims and cat the file for inspection

```bash
$ cd /etc/nginx/jwt/hs256/nested_jwt
$ ls

expired_token_nested.jwt  invalid_token_nested.jwt  valid_token_nested_1.jwt  valid_token_nested_2.jwt
```

1. Run a `curl` command inserting a valid JWT (`valid_token_nested_1.jwt`) into
   the Authorization header as a Bearer Token. We get a `HTTP 200` response
   because this JWT's sub claim is `xample@example.com`

```bash
$ curl -k -H "Authorization: Bearer `cat valid_token_nested_1.jwt`" https://localhost/products/

Thank you for requesting /products/
```

1. Run a `curl` command inserting another valid JWT (`valid_token_nested_2.jwt`)
   into the Authorization header as a Bearer Token. This time we get a `HTTP
   403` because this JWT's sub claim is `bobby@example.com` which has been
   revoked

```bash
$ curl -k -I -H "Authorization: Bearer `cat valid_token_nested_2.jwt`" https://localhost/admin

HTTP/1.1 403 Forbidden
Server: nginx/1.13.7
Date: Mon, 02 Apr 2018 16:19:44 GMT
Content-Type: application/json
Content-Length: 169
Connection: keep-alive

curl -k -H "Authorization: Bearer `cat valid_token_nested_2.jwt`" https://localhost/admin
{"status":403,"message":"Forbidden"}

```

---------

[Back to Main Menu](../README.md)