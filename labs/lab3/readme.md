# API Key Authentication

## Introduction

NGINX offers several approaches to safeguard APIs and authenticating API
clients. A standard method is API key authentication.

An API key is a simple encrypted string that identifies an application without
any principal. They safeguard access API data and are used to associate API
requests with your Clients for audit purposes, quota and billing.

API keys are a shared secret known by the client and the API gateway. An API key
is a long and complex password issued to the API client as a long‑term
credential. Creating API keys is simple – encode a random number, as in this
example.

For more details, read:
 * [Implementing
   Authentication](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-1/#implement-auth)

## Learning Objectives 

By the end of the lab, you will be able to: 
 * Enforce API Key Authentication to protect API resources

## Configure API Key Authentication

1. Enable the config `warehouse_api_apikeys.conf`

```bash
for x in /etc/nginx/api_conf.d/*.conf; do mv "$x" "${x%.conf}.conf_"; done
mv /etc/nginx/api_conf.d/warehouse_api_apikeys.conf_ /etc/nginx/api_conf.d/warehouse_api_apikeys.conf
ls /etc/nginx/api_conf.d
nginx -t && nginx -s reload
```

2.  Show API Key Authenication configured in `/etc/nginx/api/api_keys.conf` in
    the `map` block and enforced in `if` block configured in
    `/etc/nginx/api/api_gateway.conf`.

`map` block sets which API clients (`$api_client_name`) are Infrastructure
clients (`$is_infrastructure`):

```bash
$ bat /etc/nginx/api/api_keys.conf
```

Inspect the following `map` blocks:

```nginx
# API Clients
 map $http_apikey $api_client_name {
    default "";

    "7B5zIqmRGXmrJTFmKa99vcit" "client_one";
    "QzVV6y1EmQFbbxOfRCwyJs35" "client_two";
    "mGcjH8Fv6U9y3BVF9H3Ypb9T" "client_six";
}

# Infrastructure clients
map $api_client_name $is_infrastructure {
    default       0;

    "client_one"  1;
    "client_six"  1;
}
```

The `if ($is_infrastructure = 0) { #...}` checks if the varible
`$is_infrastructure` is set, if not, Nginx responds with a `HTTP 403`

 ```nginx
 $ bat /etc/nginx/api_conf.d/warehouse_api_apikeys.conf

 # ...

 location = /api/warehouse/inventory/audit {
    if ($is_infrastructure = 0) {
        return 403; # Forbidden (not infrastructure)
    }
    set $upstream inventory_service;
    rewrite ^ /_warehouse last;
}
```
Also note the "Policy Section" where API Keys are enforced across all API
endpoints:

```nginx
# Policy section
#
location = /_warehouse {
    internal;
    set $api_name "Warehouse";

    # Policy configuration here (authentication, rate limiting, logging, more...)

    if ($http_apikey = "") {
        return 401; # Unauthorized (please authenticate)
    }
    if ($api_client_name = "") {
        return 403; # Forbidden (invalid API key)
    }

    proxy_pass http://$upstream$request_uri;
}
```

1. We can see this in action with `curl` with requests using Missing, Invalid
   and, Valid API keys:

```bash
# No API key
$ curl -i -k https://localhost/api/warehouse/pricing/item001
{"status":401,"message":"Unauthorized"}

# Invalid API key
$ curl -H "apikey: thisIsInvalid" -i -k https://localhost/api/warehouse/pricing/item001
{"status":403,"message":"Forbidden"}

# Valid API key
$ curl -H "apikey: 7B5zIqmRGXmrJTFmKa99vcit" -i -k https://localhost/api/warehouse/pricing/item001
{"status": "200",...}
```

2. Specificly test the `/api/warehouse/inventory/audit` API endpoint where
   certain `is_infrastructure` API keys can access (`client_one` and
   `client_six`)

```
# No API key
$ curl -i -k https://localhost/api/warehouse/inventory/audit
{"status":401,"message":"Unauthorized"}

# Valid key NOT IN "is_infrastructure" API key (QzVV6y1EmQFbbxOfRCwyJs35)
$ curl -H "apikey: thisIsInvalid" -i -k https://localhost/api/warehouse/inventory/audit
{"status":403,"message":"Forbidden"}

# Valid IN "is_infrastructure" API key (Client One)
$ curl -H "apikey: 7B5zIqmRGXmrJTFmKa99vcit" -i -k https://localhost/api/warehouse/inventory/audit
{"status": "200",...}
```

---------

[Back to Main Menu](../README.md)