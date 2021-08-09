# JWT Authentication

## Introduction

JSON Web Tokens (JWTs, pronounced “jots”) are a compact and highly portable
means of exchanging identity information. The JWT specification has been an
essential underpinning of OpenID Connect, providing a single sign‑on token for
the OAuth 2.0 ecosystem. JWTs can also be used as authentication credentials in
their own right and are a better way to control access to web‑based APIs than
traditional API keys.

JSON Web Tokens (JWTs) are increasingly used for API authentication. Native JWT
support is exclusive to NGINX Plus.

### JWT as an API Key

A common way to authenticate an API client (the remote software client
requesting API resources) is through a shared secret, generally referred to as
an API key. A traditional API key is essentially a long and complex password
that the client sends as an additional HTTP header on each and every request.
The API endpoint grants access to the requested resource if the supplied API key
is in the list of valid keys. Generally, the API endpoint does not validate API
keys itself; instead, an API gateway handles the authentication process and
routes each request to the appropriate endpoint. Besides computational
offloading, this provides the benefits that come with a reverse proxy, such as
high availability and load balancing to a number of API endpoints.

The API gateway validates the API key by consulting a key registry before
passing the request to the API endpoint: ![JWT API Gateway
Tradtional](media/jwt-api-gateway-traditional-key.png)

Using JWT as the API key provides a high‑performance alternative to traditional
API keys, combining best‑practice authentication technology with a
standards‑based schema for exchanging identity attributes: ![JWT API Gateway
NGINX](media/jwt-nginx-plus-jwt-key.png)

### NGINX Plus JWT 

NGINX Plus supports validating JWTs directly, and so Nginx functions as an
excellent API gateway, providing a frontend to an API endpoint and using JWT to
authenticate client applications.

#### Nginx Plus supports the following cryptographic algorithms:

NGINX Plus supports both types of JWT listed below ([See our
docs]((https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-jwt-authentication/))
for the most up-to-date list)
 * **JSON Web Signature (JWS)** - the contents of JWT is digitally signed. The
   following algorithms can be used for signing:
    * HS256, HS384, HS512
    * RS256, RS384, RS512
    * ES256, ES384, ES512
    * EdDSA (Ed25519 and Ed448 signatures)

 * **JSON Web Encryption (JWE)** - the contents of JWT is encrypted. The
   following content encryption algorithms (the “enc” field of JWE header) are
   supported:
    * A128CBC-HS256, A192CBC-HS384, A256CBC-HS512
    * A128GCM, A192GCM, A256GCM
    * The following key management algorithms (the “alg” field of JWE header)
      are supported:
      * A128KW, A192KW, A256KW
      * A128GCMKW, A192GCMKW, A256GCMKW
      * dir - direct use of a shared symmetric key as the content encryption key

 The Nginx configuration below demonstrates validation of JSON Web Token passed
 with a request in various. By default, JWT is passed in the Authorization
 header as a Bearer Token, but JWT may also be passed as a cookie or a part of a
 query string:

```nginx
    # Bearer Token JWT
    # By default, JWT is passed in the "Authorization" header as a Bearer Token
    location = /jwt/hs256 {
        proxy_pass http://echo_http;
        auth_jwt "Example hs256 API"; # JWT in "Authorization" header as a Bearer Token # (Default)
        auth_jwt_key_file jwk/hs256/secrets.jwk;
    }

    # Cookie JWT
    location ~ /jwt/hs256/cookie {
        proxy_pass http://echo_http;
        auth_jwt "Example hs256 API"  token=$cookie_auth_token; # JWT in cookie
        auth_jwt_key_file jwk/hs256/secrets.jwk;
    }

    # URL query argurments JWT
    location ~ /jwt/hs256/args {
        proxy_pass http://echo_http;
        auth_jwt "Example hs256 API" token=$arg_apijwt; # JWT in URL query
        auth_jwt_key_file jwk/hs256/secrets.jwk;
    }
```

For more details read:

 * [Authenticating API Clients with JWT and NGINX
   Plus](https://www.nginx.com/blog/authenticating-api-clients-jwt-nginx-plus/)
 * [Controlling Access to Specific
   Methods](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-2-protecting-backend-services/#access-control-methods)
 * Nginx Module:
   [`ngx_http_auth_jwt_module`](http://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html)


## Learning Objectives 

By the end of the lab you will be able to: 
 * Authenticating API Clients with JWT

### Inspect the example JWTs

As a shortcut to reproduce these demos, we will use prepared JWTs in text files
and will insert them in our `curl` commands.

You can find the JWT files in the following directories:

 * HS256 `jwt/hs256`
 * HS384 `jwt/hs384`
 * HS512 `jwt/hs512`
 * RS256 `jwt/rs256`
 * RS384 `jwt/rs384`
 * RS512 `jwt/rs512`
 * ES256 `jwt/es256`
 * ES384 `jwt/es384`

You can use [jwt.io](https://jwt.io) or a simple bash script to decode the JWT
payload and inspect details about the token, such as the `exp` field for expiry
in epoch time

**Note:** When validating RSA (`RS256`, `RS384` and `RS512`) and ECDSA (`ES256`
and `ES384`) asymmetric encrypted JWT using [jwt.io](https://jwt.io) you require
the certificate and private key to decode and verify


1. Enable the config `warehouse_api_jwt.conf`

```bash
for x in /etc/nginx/api_conf.d/*.conf; do mv "$x" "${x%.conf}.conf_"; done
mv /etc/nginx/api_conf.d/warehouse_api_jwt.conf_ /etc/nginx/api_conf.d/warehouse_api_jwt.conf
ls /etc/nginx/api_conf.d
nginx -t && nginx -s reload
```

1. Let's inspect **HS256** based JWT. This is a valid JWT with an exp of
   `1545091200` (`Tuesday, December 17, 2018 12:00:00 AM`)

```bash
# Move into folder with the JWTs and cat the file for inspection
$ cd /etc/nginx/jwt/hs256

# Print the token
$ cat token.jwt
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NDUwOTEyMDAsIm5hbWUiOiJDcmVhdGUgTmV3IFVzZXIiLCJzdWIiOiJjdXNlciIsImduYW1lIjoid2hlZWwiLCJndWlkIjoiMTAiLCJmdWxsTmFtZSI6IkpvaG4gRG9lIiwidW5hbWUiOiJqZG9lIiwidWlkIjoiMjIyIiwic3VkbyI6dHJ1ZSwiZGVwdCI6IklUIiwidXJsIjoiaHR0cDovL3NlY3VyZS5leGFtcGxlLmNvbSJ9.YYQCNvzj17F726QvKoIiuRGeUBl_xAKj62Zvc9xkZb4

$ JWT=token.jwt
$ jq -R 'split(".") | .[0],.[1] | @base64d | fromjson' <<< $(cat "${JWT}")

{
  "alg": "HS256",
  "typ": "JWT"
}
{
  "exp": 1545091200,
  "name": "Create New User",
  "sub": "cuser",
  "gname": "wheel",
  "guid": "10",
  "fullName": "John Doe",
  "uname": "jdoe",
  "uid": "222",
  "sudo": true,
  "dept": "IT",
  "url": "http://secure.example.com"
}
```

1. Let's inspect another **HS256** based JWT. This is a valid JWT with an exp of
   `2689248651` (`Sunday, March 21, 2055 1:30:51 PM`)

```
$ cat valid_token.jwt

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjI2ODkyNDg2NTEsIm5hbWUiOiJib2JieSBkaWdpdGFsIiwic3ViIjoiY3VzZXIiLCJnbmFtZSI6IndoZWVsIiwiZ3VpZCI6IjEwIiwiZnVsbE5hbWUiOiJib2JieSBkaWdpdGFsIiwidW5hbWUiOiJiZGlnaXRhbCIsInVpZCI6IjIyMiIsInN1ZG8iOnRydWUsImRlcHQiOiJJVCIsInVybCI6Imh0dHA6Ly93d3cuZXhhbXBsZS5jb20ifQ.nYLDFywgByZsH5Lhxje0B0j3uGvvpFd0bJsGchpNAuc

$ JWT=valid_token.jwt
$ jq -R 'split(".") | .[0],.[1] | @base64d | fromjson' <<< $(cat "${JWT}")

{
  "alg": "HS256",
  "typ": "JWT"
}
{
  "exp": 2689248651,
  "name": "bobby digital",
  "sub": "cuser",
  "gname": "wheel",
  "guid": "10",
  "fullName": "bobby digital",
  "uname": "bdigital",
  "uid": "222",
  "sudo": true,
  "dept": "IT",
  "url": "http://www.example.com"
}
```

3. There are other tokens you can inspect in this directory, such as
   `expired_token.jwt` and `invalid_token.jwt`

## Configure NGINX Plus as an Authenticating API Gateway

1. Inspect the config, `warehouse_api_jwt.conf`, and see how NGINX can find the
   `jwt` in the `Authorization: Bearer` **header**, **Cookie**, and *URL
   parameter* 

```bash
$ bat /etc/nginx/api_conf.d/warehouse_api_jwt.conf
```

Follow the steps below to validate a JWT with Nginx Plus. For brevity, we show
only show the expected output of the `HS256` token validation

### Validating JWTs

In this example we will again use `HS256` based JWTs, run the steps outlined
below using `HS256` signed keys and the `HS256` test URL endpoints. 

#### Validate a valid JWT in Bearer Token

1. Run a `curl` command inserting the valid JWT (`valid_token.jwt`) into the
   `Authorization` header as a Bearer Token

```bash
$ curl -k -H "Authorization: Bearer `cat valid_token.jwt`" https://localhost/jwt/hs256

Thank you for requesting /jwt/hs256
```

1. To see the response of a expired JWT, Run a curl command inserting the
   expired JWT (`expired_token.jwt`) into the `Authorization` header as a Bearer
   Token

You should get a `HTTP 401` and a error message containing `expired JWT token`
should be logged in the `error.log` file

```bash
$ curl -k -H "Authorization: Bearer `cat expired_token.jwt`" https://localhost/jwt/hs256
{"status":401,"message":"Unauthorized"}

$ tail -f 10 /var/log/nginx/error.log

...
2018/03/21 15:13:23 [info] 2870894#2870894: *1935 expired JWT token, client: 159.64.72.245, server: jwt.example.com, request: "GET /jwt/hs256 HTTP/1.1", host: "jwt.example.com"
...
```

1. Similarly, NGINX will respond with an `HTTP 401` when attempting to validate
   an invalid token. Run a curl command inserting the invalid JWT
   (`invalid_token.jwt`) into the `Authorization` header as a Bearer Token

You should get an `HTTP 401`, and a error message containing `JWT HS validation
failed` should get logged in the `error.log` file

```bash
$ curl -k -H "Authorization: Bearer `cat invalid_token.jwt`" https://localhost/jwt/hs256
{"status":401,"message":"Unauthorized"}

$ tail -f 10 /var/log/nginx/error.log

...
2018/03/21 16:03:43 [info] 2876917#2876917: *1994 JWT HS validation failed, client: 159.35.72.245, server: jwt.example.com, request: "GET /jwt/hs256 HTTP/1.1", host: "jwt.example.com"
...
```

#### Validate JWT in Cookie

1. Run a `curl` command inserting the JWT (`valid_token.jwt`) into the
   `auth-token` cookie

```bash
$ curl -v -k -i -b "auth_token=`cat valid_token.jwt`" https://localhost/jwt/hs256/cookie

*   Trying 159.65.72.245...
* Connected to jwt.example.com (159.65.72.245) port 80 (#0)
> GET /jwt/hs256/cookie HTTP/1.1
> Host: jwt.example.com
> User-Agent: curl/7.47.0
> Accept: */*
> ....#SNIPPED#....
> Cookie: auth_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjI2ODkyNDg2NTEsIm5hbWUiOiJib2JieSBkaWdpdGFsIiwic3ViIjoiY3VzZXIiLCJnbmFtZSI6IndoZWVsIiwiZ3VpZCI6IjEwIiwiZnVsbE5hbWUiOiJib2JieSBkaWdpdGFsIiwidW5hbWUiOiJiZGlnaXRhbCIsInVpZCI6IjIyMiIsInN1ZG8iOnRydWUsImRlcHQiOiJJVCIsInVybCI6Imh0dHA6Ly93d3cuZXhhbXBsZS5jb20ifQ.nYLDFywgByZsH5Lhxje0B0j3uGvvpFd0bJsGchpNAuc
>
< HTTP/1.1 200 OK
< Server: nginx/1.13.7
< Date: Wed, 21 Mar 2018 16:10:29 GMT
< Content-Type: text/plain
< Content-Length: 43
< Connection: keep-alive
< X-Whom: WEB-ECHO
<
Thank you for requesting /jwt/hs256/cookie
* Connection #0 to host jwt.example.com left intact
```

#### Validate JWT in a query parameter

1. Run a `curl` command inserting the JWT (`valid_token.jwt`) into the the query
   parameter value of `apijwt`in the URL

```bash
$ curl -k -i "https://localhost/jwt/hs256/args/index.html?apijwt=`cat valid_token.jwt`"

HTTP/1.1 200 OK
Server: nginx/1.13.7
Date: Wed, 21 Mar 2018 16:00:06 GMT
Content-Type: application/json
Content-Length: 403
Connection: keep-alive
X-Whom: WEB-ECHO
```

#### Validate other types of JWTs

We can use the same steps above to validate other types of signed JWTs, run the
same steps outlined in the instructions above but this time using the other
signed keys provided in */etc/nginx/jwt/*

1. For example, to test validation of `HS384` based JWTs, we can use the `HS384`
   signed keys and the `HS384` test URL endpoints.

```bash
# Directory of hs384 JWTs
$ cd /etc/nginx/jwt/hs384
$ ls
expired_token.jwt  invalid_token.jwt  token.jwt  valid_token.jwt
```

2.  Run `curl` commands using the appropriate JWT and URI Endpoint. Note the URI
    Path includes **hs384** for this example using **hs384** signed JWT

```bash
#### Validate a JWT in Bearer Token
# Change the test JWT to `expired_token.jwt`, `invalid_token.jwt`, `token.jwt` or `valid_token.jwt`
$ curl -k -H "Authorization: Bearer `cat valid_token.jwt`" https://localhost/jwt/hs384

#### Validate a JWT in Cookie
# Change the test JWT to `expired_token.jwt`, `invalid_token.jwt`, `token.jwt` or `valid_token.jwt`
$ curl -v -k -i -b "auth_token=`cat valid_token.jwt`" https://localhost/jwt/hs384/cookie

#### Validate a JWT in a query parameter
# Change test JWT to `expired_token.jwt`, `invalid_token.jwt`, `token.jwt` or `valid_token.jwt`
$ curl -k "https://localhost/jwt/hs384/args/index.html?apijwt=`cat valid_token.jwt`"
```
---------

[Back to Main Menu](../README.md)