# Nested JWT Claims and Array Data

## Introduction

NGINX Plus support nested JWT claims and array data and this provides more
flexibility and greater security when validating JWTs.

A JWT payload can contain additional "nested" information about the user, such
as group membership, which can be used to authorize access to a resource. This
helps avoid the need to query a database multiple times to authorize user
requests. What a nested object and array look like:

 * Example nested object - `"name_of_nested_object": { "name":"John", "age":30,
   "license":true }`
 * Example array -  `"name_of_array":[ "Ford", "BMW", "Fiat" ]`

The example JWT below, has a "nested claim" object called **attributes** and an
array called **groups**.

```json
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

For more details read:

 * [Authenticating API Clients with JWT and NGINX
   Plus](https://www.nginx.com/blog/authenticating-api-clients-jwt-nginx-plus/)
 * [Controlling Access to Specific
   Methods](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-2-protecting-backend-services/#access-control-methods)
 * Nginx Module:
   [`ngx_http_auth_jwt_module`](http://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html)


## Learning Objectives 

By the end of the lab you will be able to: 
 * Authenticating API Clients with Nested JWT

## Inspect Nested NGINX configs saving JWT Claims and Array Data as varibles

1. Inspect `api_jwt.conf` and understand the following Logic Flow

 1. Using the `auth_jwt_claim_set` directive, we set the `$jwt_groups` variable
    to the groups array defined in the JWT. The values in the array are
    separated by commas and assigned to `$jwt_groups`.
 2. Using the `map` directive, we search through the list of groups for the
    "`Administrator`" keyword. If it’s present, the user is deemed an
    Administrator and `$isAdmin` is set to `1`. Otherwise `$isAdmin` is set to
    ``.
 3. The location block for the `/admin` URI checks the $isAdmin variable. If
    `0`, Nginx returns status code `403 Forbidden` back to the client.

```
$ cat /etc/nginx/api/api_jwt.conf
```

```nginx
# Set variables from the JWT claim parameter identified by key names
# Name matching starts from the top level of the JSON tree.
# For arrays, the variable keeps a list of array elements separated by commas.

auth_jwt_claim_set $jwt_groups groups; # Save "group" array values in a variable as comma-separated values
auth_jwt_claim_set $jwt_real_name attributes name; #  Save nested object 2 levels deep, attributes > name in a variable

# Map - Check group array in JWT for "Administrator" to determine if the user is an Admin
map $jwt_groups $isAdmin {
  "~\bAdministrator\b" 1; # Appears within word boundaries (\b) in varible with a CSV (Comma-separated values) list
  default              0;
}

server {

    # ...

    # Admin Only Access: Administrator group only
    location = /admin {
        if ( $isAdmin = 0 ) {
            return 403; # Forbidden
        }
        proxy_pass http://info_http;

        # JWT in "Authorization" header as a Bearer Token (Default)
        # Using HS256 cryptographic algorithm
        auth_jwt "My secure site";
        auth_jwt_key_file jwk/hs256/secrets.jwk;

        # We can also pass Values in the JWT
        proxy_set_header X-RealName $jwt_real_name; # Pass real name as header
        proxy_set_header X-Subject  $jwt_claim_sub; # L1 claims set automatically
    }
}
```

## Inspect JWT Claims and Array Data 


1. To reproduce these demos, we will use prepared JWTs in text files and will
   insert them in our `curl` commands. You can find the nested JWT files in the
   following directory, `/etc/nginx/jwt/hs256/nested_jwt`

There are two valid tokens we will test against: `valid_token_nested_1.jwt` and
`valid_token_nested_2.jwt`, where `valid_token_nested_1.jwt` contains the group
"**Administrator**" and should be authenicated, while `valid_token_nested_2.jwt`
has "**Intern**" and will be denied access. Lets inspect
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

### Nested JWT claims Validation

1. Enable the config `warehouse_api_simple.conf`

```bash
for x in /etc/nginx/api_conf.d/*.conf; do mv "$x" "${x%.conf}.conf_"; done
mv /etc/nginx/api_conf.d/warehouse_api_jwt.conf_ /etc/nginx/api_conf.d/warehouse_api_jwt.conf
ls /etc/nginx/api_conf.d
nginx -t && nginx -s reload
```
In this example we will validate a nested JWT claims by checking the user is in
the Administrator Group:

1. Move into folder with the nested JWT claims and cat the file for inspection

```bash
$ cd /etc/nginx/jwt/hs256/nested_jwt
$ ls

expired_token_nested.jwt  invalid_token_nested.jwt  valid_token_nested_1.jwt  valid_token_nested_2.jwt
```

1. Run a `curl` command inserting a valid JWT (`valid_token_nested_1.jwt`) into
   the Authorization header as a Bearer Token.

1. We get a `HTTP 200` response because the User request does identify herself
   as an Administrator, i.e. the User request with with this token has
   "Administrator" listed in the "groups" array

```bash
$ curl -k -H "Authorization: Bearer `cat valid_token_nested_1.jwt`" https://localhost/admin
{
                "data": [
                    {
                    "status": "200",
                    "server_address": "server_addr:8833",
                    "server_name":  "1fe71e5f3f14",
                    "date": "01/May/2020:13:08:46 +0000",
                    "uri": "/admin",
                    "request_id": "866d1a3c61a4848bcf667db2866d7a38"
                    }
                ]

```

1. Run a `curl` command inserting another valid JWT (`valid_token_nested_2.jwt`)
   into the Authorization header as a Bearer Token. This time we get a `HTTP
   403` because the User request does not identify herself as an Administrator,
   i.e. the User request with with this token does not have "Administrator"
   listed in the "groups" array

```bash
$ curl -k -H "Authorization: Bearer `cat valid_token_nested_2.jwt`" https://localhost/admin
{"status":403,"message":"Forbidden"}
```


---------

[Back to Main Menu](../README.md)