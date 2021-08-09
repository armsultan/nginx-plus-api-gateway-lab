# Routing - Choosing Broad vs. Precise Definition for APIs

## Introduction

There are two approaches to API definition – **broad** and **precise**. The most
suitable approach for each API depends on the API’s security requirements and
whether the backend services should handle invalid URIs.

For more details, read: 
 * [Choosing Broad vs. Precise Definition for
   APIs](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-1/)

## Learning Objectives 

By the end of the lab, you will be able to: 
 * Configure two approaches to API definition – broad and precise

### Broad Definitions

1. Enable the config: `warehouse_api_simple.conf`

```bash
for x in /etc/nginx/api_conf.d/*.conf; do mv "$x" "${x%.conf}.conf_"; done
mv /etc/nginx/api_conf.d/warehouse_api_simple.conf_ /etc/nginx/api_conf.d/warehouse_api_simple.conf
ls /etc/nginx/api_conf.d
nginx -t && nginx -s reload
```

1. Inspect the simple rouing configurations in `warehouse_api_simple.conf`

```bash
bat /etc/nginx/api_conf.d/warehouse_api_simple.conf
```

1. show curl response of `/api/warehouse/inventory` and
   `/api/warehouse/pricing`:

```bash
# inventory api
$ curl -i -k https://localhost/api/warehouse/inventory

# pricing api
$ curl -i -k https://localhost/api/warehouse/pricing

# These broad requests also work:
$ curl -i -k https://localhost/api/warehouse/inventory
$ curl -i -k https://localhost/api/warehouse/inventory/
$ curl -i -k https://localhost/api/warehouse/inventory/foo
$ curl -i -k https://localhost/api/warehouse/inventoryfoo
$ curl -i -k https://localhost/api/warehouse/inventoryfoo/bar/
```

### Precise Definitions

A more precise approach enables the API gateway to understand the API’s full URI
space by explicitly defining the URI path for each available API resource.
Taking the precise approach, the following configuration for URI routing in the
Warehouse API uses a combination of:

 * (`=`) exact matching, and 
 *  (`~`) regular expressions to define each and every valid URI.

More details on the modifiers below will cause the associated location block to
be interpreted as follows:

 * **(none)**: If no modifiers are present, the location is interpreted as a
   prefix match. This means that the location given will be matched against the
   beginning of the request URI to determine a match. *` =` - If an equal sign
   is used, this block will be considered a match if the request URI exactly
   matches the location given.
 * `~` - If a tilde modifier is present, this location will be interpreted as a
   case-sensitive regular expression match.
 * `~*` - If a tilde and asterisk modifier is used, the location block will be
   interpreted as a case-insensitive regular expression match.
 * `^~` - If a carat and tilde modifier is present, and if this block is
   selected as the best non-regular expression match, regular expression
   matching will not take place.


1. Enable the config: `warehouse_api_precise.conf`

```bash
for x in /etc/nginx/api_conf.d/*.conf; do mv "$x" "${x%.conf}.conf_"; done
mv /etc/nginx/api_conf.d/warehouse_api_precise.conf_ /etc/nginx/api_conf.d/warehouse_api_precise.conf
ls /etc/nginx/api_conf.d
nginx -t && nginx -s reload
```

1. Show simple rouing configurations in `warehouse_api_precise.conf`

```bash
$ bat /etc/nginx/api_conf.d/warehouse_api_precise.conf
```

1. Use `curl` to show valid endpoints:

**Valid URIs:**

 * `/api/warehouse/inventory`
 * `/api/warehouse/inventory/shelf/foo`
 * `/api/warehouse/inventory/shelf/foo/box/bar`
 * `/api/warehouse/inventory/shelf/-/box/-`
 * `/api/warehouse/pricing/baz`

These `curl` commands result in `HTTP 200`:

```bash
$ curl -i -k https://localhost/api/warehouse/inventory
$ curl -i -k https://localhost/api/warehouse/inventory/shelf/foo
$ curl -i -k https://localhost/api/warehouse/inventory/shelf/foo/box/bar
$ curl -i -k https://localhost/api/warehouse/inventory/shelf/-/box/-
$ curl -i -k https://localhost/api/warehouse/pricing/baz
```

1. Use `curl` to show invalid endpoints:

**Invalid URIs:**

 * `/api/warehouse/inventory/`
 * `/api/warehouse/inventoryfoo`
 * `/api/warehouse/inventory/shelf`
 * `/api/warehouse/inventory/shelf/foo/bar`
 * `/api/warehouse/pricing`
 * `/api/warehouse/pricing/baz/`

These `curl` commands result in `HTTP 400`:
```bash
$ curl -i -k https://localhost/api/warehouse/inventory/
$ curl -i -k https://localhost/api/warehouse/inventoryfoo
$ curl -i -k https://localhost/api/warehouse/inventory/shelf
$ curl -i -k https://localhost/api/warehouse/inventory/shelf/foo/bar
$ curl -i -k https://localhost/api/warehouse/pricing
$ curl -i -k https://localhost/api/warehouse/pricing/baz/
```

---------

[Back to Main Menu](../README.md)