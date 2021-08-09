# Controlling Request Sizes

## Introduction

HTTP APIs commonly use the request body to contain instructions and data for the
backend API service to process. This is true of XML/SOAP APIs as well as
JSON/REST APIs. Consequently, the request body can pose an attack vector to the
backend API services, which may be vulnerable to buffer overflow attacks when
processing substantial request bodies.

By default, NGINX rejects requests with bodies larger than 1 MB. This can be
increased for APIs that specifically deal with large payloads such as image
processing, but for most APIs, we set a lower value.

For more details read: [Controlling Request
Sizes](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-2-protecting-backend-services/#request-size)

## Learning Objectives 

By the end of the lab you will be able to: 
 * Control Request Sizes, Increase or decrease the default 1MB limit

## Configure Specific Request Sizes

1. Enable the config `warehouse_api_bodysize.conf`.

```bash
for x in /etc/nginx/api_conf.d/*.conf; do mv "$x" "${x%.conf}.conf_"; done
mv /etc/nginx/api_conf.d/warehouse_api_bodysize.conf_ /etc/nginx/api_conf.d/warehouse_api_bodysize.conf
ls /etc/nginx/api_conf.d
nginx -t && nginx -s reload
```

1. Inspect he `warehouse_api_bodysize.conf` and the `client_max_body_size`
   directives limiting the allowed upload size: 

```
$ bat /etc/nginx/api_conf.d/warehouse_api_bodysize.conf

#...
client_max_body_size 16k;
```

1. Lets create adummy file that is over the `client_max_body_size` threshold

```
# Create a dummy 1 MB file:
$ truncate -s 1M test.json
```

1. Now test the Request Size rule by `POST`ing the test file to the protected
   API endpoint

```bash
$ curl -k -X POST --data-binary "@test.json" https://localhost/api/warehouse/pricing/item00

{"status":413,"message":"Payload too large"}
```
---------

[Back to Main Menu](../README.md)