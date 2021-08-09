# Validating Request Bodies

## Introduction

The following use case is one of several for the NGINX JavaScript module. For a
complete list, see [Use Cases for the NGINX JavaScript
Module](https://www.nginx.com/blog/harnessing-power-convenience-of-javascript-for-each-request-with-nginx-javascript-module/#Use-Cases-for-the-NGINX-JavaScript-Module)

In addition to being vulnerable to buffer overflow attacks with large request
bodies, backend API services can be susceptible to bodies that contain invalid
or unexpected data. For applications that require correctly formatted JSON in
the request body, we can use the [NGINX JavaScript
module](https://www.nginx.com/blog/harnessing-power-convenience-of-javascript-for-each-request-with-nginx-javascript-module)
to verify that JSON data is parsed without error before proxying it to the
backend API service.

With the [JavaScript module
installed](https://docs.nginx.com/nginx/admin-guide/dynamic-modules/nginscript/),
we use the js_import directive to reference the file containing the JavaScript
code for the function that validates JSON data.

### A Note about the $request_body Variable


For more details read: [Controlling Request
Sizes](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-2-protecting-backend-services/#request-size)

## Learning Objectives 

By the end of the lab you will be able to: 
 * Use the NGINX JavaScript Module to Validating Request Bodies

## Configure Validating Request Bodies using NGINX JavaScript

1. Enable the config `warehouse_api_jsonbody.conf`.

```bash
for x in /etc/nginx/api_conf.d/*.conf; do mv "$x" "${x%.conf}.conf_"; done
mv /etc/nginx/api_conf.d/warehouse_api_jsonbody.conf_ /etc/nginx/api_conf.d/warehouse_api_jsonbody.conf
ls /etc/nginx/api_conf.d
nginx -t && nginx -s reload
```

1. Inspect the `nginx.conf` file where we enable the NGINX javascript module

```bash
$ bat /etc/nginx/nginx.conf
```

Find the following line

```nginx
# Load Modules
# NGINX Javascript
load_module /etc/nginx/modules/ngx_http_js_module.so; 
```

1. Now, Inspect the `api_gateway.conf` file where we use the
   [`js_import`](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_import)
   directive to reference the file containing the JavaScript code for the
   function that validates JSON data.

```bash
$ bat /etc/nginx/api/api_gateway.conf
```

Find the following lines in the file:

```nginx
js_import json_validation.js;
js_set $json_validated json_validation.parseRequestBody;
```

 * The [`js_set`](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_set)
   directive defines a new variable, `$json_validated`, which is evaluated by
   calling the `parseRequestBody` function.

1. Now inspect the NGINX Javascript file, `json_validation.js`:

```bash
$ bat /etc/nginx/js/json_validator.js
```

Inspect this NGINX JavaScript file: 

```js
export default { parseRequestBody };

function parseRequestBody(r) {
    try {
        if (r.variables.request_body) {
            JSON.parse(r.variables.request_body);
        }
        return r.variables.upstream;
    } catch (e) {
        r.error('JSON.parse exception');
        return '127.0.0.1:10415'; // Address for error response
    }
}
```

 * The `parseRequestBody` function attempts to parse the request body using the
   `JSON.parse` method.
 * If parsing succeeds, the name of the intended upstream group for this request
   is returned.
 * If the request body cannot be parsed (causing an exception), a local server
   address is returned.
 * The `return` directive populates the `$json_validated` variable so that we
   can use it to determine where to send the request.

1. Now inspect `warehouse_api_jsonbody.conf`.

```bash
$ bat /etc/nginx/api_conf.d/warehouse_api_jsonbody.conf
```

Find the following `location` block:

```nginx

#...
    location /api/warehouse/pricing {
        set $upstream warehouse_pricing;
        mirror /_NULL;                    # Force early read
        client_body_in_single_buffer on;  # Minimize memory copy operations on request body
        client_body_buffer_size      16k; # Largest body to keep in memory (before writing to file)
        client_max_body_size 16k;
        proxy_pass http://$json_validated$request_uri;
    }
#...
```

  In this URI routing section of the Warehouse API, we modify the `proxy_pass`
  directive:

   * It passes the request to the backend API service as in the Warehouse API
     configurations discussed in previous sections, but now uses the
     `$json_validated` variable as the destination address. 
   * If the client body was successfully parsed as `JSON` then we proxy to the
     upstream group defined.
   * If, however, there was an exception, we use the returned value of
     `127.0.0.1:10415` to send an error response to the client (We see this in a
     following step)

1. **Please Note** about the `$request_body` Variable:

 * The JavaScript function `parseRequestBody` uses the `$request_body` variable
   to perform JSON parsing. However, NGINX does not populate this variable by
   default, and simply streams the request body to the backend without making
   intermediate copies. 
 * By using the `mirror` directive inside the URI routing section we create a
   copy of the client request, and consequently populate the `$request_body`
   variable.
 * The  `client_body_buffer_size` and `client_max_body_size` control how NGINX
   handles the request body internally. This improves overall performance by
   minimizing disk I/O operations, but at the expense of additional memory
   utilization. For most API gateway use cases with small request bodies this is
   a good compromise.

As mentioned, the mirror directive creates a copy of the client request. Other
than populating `$request_body`, we have no need for this copy so we send it to
a "dead end" location (`/_NULL`) that we define in the server block in the
top‑level API gateway configuration. We can see this in `api_gateway.conf`:

```bash
$ bat /etc/nginx/api/api_gateway.conf
```

1. Lastly, we handle error response to the client in this small server block, in
   this example a `HTTP 415 - Unsupported media type` Error is handled by NGINX
   and returns the custom error reponse in the `api_json_errors.conf` file

```bash
bat /etc/nginx/api/api_gateway.conf
```

Find the following `location` block:

```nginx
# Dummy location used to populate $request_body for JSON validation
location = /_NULL {
    internal;
    return 204;
}

```

Find the following `server` block:

```nginx
server {
    listen 127.0.0.1:10415;
    return 415; # Unsupported media type
    include api_json_errors.conf;
}
```

To review the custom HTTP 415 response, inspect the `api_json_errors.conf`
again:


```bash
bat /etc/nginx/api/api_json_errors.conf | grep 415

error_page 415 = @415;
location @415 { return 415 '{"status":415,"message":"Unsupported media type"}\n'; }
```

## Test Request Body validation using NGINX JavaScript

With this complete configuration in place, NGINX proxies requests to the backend
API service only if they have correctly formatted JSON bodies.

1. Run some `curl` commands to seet Request Body validation in action

```
# Validated, everything is good!
$ curl -k -iX POST -d '{"sku":"item002","price":85.00}' \
    https://localhost/api/warehouse/pricing

{"data":[{"status":"200", #...}]}

# Invalid request, reject with HTTP 415
curl -k -X POST -d 'item002=85.00' https://localhost/api/warehouse/pricing
{"status":415,"message":"Unsupported media type"}

```
---------

[Back to Main Menu](../README.md)