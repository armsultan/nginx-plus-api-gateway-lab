# Rate Limiting

## Introduction

Unlike browser‑based clients, individual API clients can place huge loads on
your APIs, even to the extent of consuming such a high proportion of system
resources that other API clients are effectively locked out. Not only malicious
clients pose this threat: a misbehaving or buggy API client might enter a loop
that overwhelms the backend. To protect against this, we apply a rate limit to
ensure fair use by each client and to protect the resources of the backend
services.

NGINX can apply rate limits based on any attribute of the request. The client IP
the address is typically used, but when authentication is enabled for the API,
the authenticated client ID is a more reliable and accurate attribute.

Rate limits themselves are defined in the top‑level API gateway configuration
file and can then be applied globally, on a per‑API basis, or even per URI.

For more details, read:
 * [Rate
   limiting](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-2-protecting-backend-services/#rate-limit)

## Learning Objectives 

By the end of the lab, you will be able to: 
 * Enforce rate limiting rules to protect API resources

## Configure Rate Limiting

1. Enable the config `warehouse_api.conf`

```bash
for x in /etc/nginx/api_conf.d/*.conf; do mv "$x" "${x%.conf}.conf_"; done
mv /etc/nginx/api_conf.d/warehouse_api.conf_ /etc/nginx/api_conf.d/warehouse_api.conf
ls /etc/nginx/api_conf.d
nginx -t && nginx -s reload
```

1. Let's inspect Rate limiting definitions set as `limit_req_zone` parameters in
   `api_gateway.conf` and the location block in `warehouse_api.conf` where it is
   applied:

The rate limiting rules defined:

```
$ bat /etc/nginx/api/api_gateway.conf

limit_req_zone $binary_remote_addr zone=client_ip_10rs:1m rate=10r/s;
limit_req_zone $http_apikey        zone=apikey_50rs:1m   rate=50r/s;
```

The location blocks (API endpoints) where ratelimiting is enforced:

```
$ bat /etc/nginx/api_conf.d/warehouse_api.conf

# ...

location = /_warehouse {
    internal;
    set $api_name "Warehouse";

    limit_req zone=client_ip_10rs;
    #limit_req zone=apikey_50rs;
    limit_req_status 429;

    proxy_pass http://$upstream$request_uri;
}
```

1. Run `vegeta` on the NGINX host or from a remote machine to show a rate limit
   of **10 requests per second** based on source IP address:

```bash
# vegeta
$ echo "GET https://localhost/api/warehouse/pricing" | vegeta attack -duration=1s -insecure | tee results.bin | vegeta report
```

You should see the report showing only ten requests result in `HTTP 200`'s per
second. All Other responses are denied, i.e., `HTTP 429`:

```bash
#...
Status Codes  [code:count]             429:240  200:10
```

1. You can also run `vegeta` to show our second rate-limiting rule, a limit of
   200 requests per sec based on the API key. But first, you need to configure
   the configuration file, `/etc/nginx/api_conf.d/warehouse_api.conf` to use the
   other defined rate-limiting rule. We can do this by commenting out `limit_req
   zone=client_ip_10rs;` and uncommenting `limit_reqzone=apikey_50rs;`


```
$ vi /etc/nginx/api_conf.d/warehouse_api.conf

    ...
    #limit_req zone=client_ip_10rs;
    limit_req zone=apikey_50rs;
```

1. Then reload Nginx with the new configuration:

```bash
nginx -t && nginx -s reload
```

1. Now run `vegeta` to show the rate limit of 50 requests per sec based on the
   API key. Note that not all host machines running these commands can output
   the sufficient load to demonstrate the test:


```bash
# vegeta
$ echo "GET https://localhost/api/warehouse/pricing" | vegeta attack \
        -duration=1s \
        -rate=250  \
        -workers=20 \
        -insecure \
        -header "apikey: 7B5zIqmRGXmrJTFmKa99vcit" | tee results.bin | vegeta report
```

 You should see the report showing only 200 requests result in `HTTP 200`'s per
 second. All Other responses are denied, i.e, `HTTP 429`:


```bash
#...
Status Codes  [code:count]             429:50  200:50
```


---------

[Back to Main Menu](../README.md)