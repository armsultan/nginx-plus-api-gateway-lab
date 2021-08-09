# Enforcing Specific Request Methods

## Introduction

With RESTful APIs, the HTTP method (or verb) is an important part of each API
call and very significant to the API definition. Take the pricing service of our
Warehouse API as an example:

The HTTP method (or verb) is an important part of RESTful API calls and
significant to the API definition. Take the pricing service of our Warehouse API
as an example:

 * `GET /api/warehouse/pricing/item001`     returns the price of `item001`
 * `PATCH /api/warehouse/pricing/item001`   changes the price of `item001`


For more details read: [Enforcing Specific Request
Methods](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-2-protecting-backend-services/#enforce-methods)

## Learning Objectives 

By the end of the lab you will be able to: 
 * Enforce Specific Request Methods for API resources

## Configure Specific Request Method enforcement

1. Enable the config `warehouse_api.conf`.

```bash
for x in /etc/nginx/api_conf.d/*.conf; do mv "$x" "${x%.conf}.conf_"; done
mv /etc/nginx/api_conf.d/warehouse_api.conf_ /etc/nginx/api_conf.d/warehouse_api.conf
ls /etc/nginx/api_conf.d
nginx -t && nginx -s reload
```

1. Inspect he `warehouse_api.conf` and the `limit_except` directives limiting
    the allowed HTTP Methods:

```
$ bat /etc/nginx/api_conf.d/warehouse_api.conf

#...
location /api/warehouse/pricing {
    limit_except GET PATCH {
        deny all;
    }
#...
}
  
  location /api/warehouse/inventory {
      limit_except GET {
          deny all;
      }
#...
  }
```

1. Lets see HTTP methods access controls in action

```bash
# GET
$ curl -k https://localhost/api/warehouse/pricing/item001
{"status": "200",...}

# DELETE
$ curl -k -X DELETE https://localhost/api/warehouse/pricing/item001
{"status":405,"message":"Method not allowed"}
```

---------

[Back to Main Menu](../README.md)