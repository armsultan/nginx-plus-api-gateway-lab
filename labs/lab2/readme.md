# Custom JSON error pages

## Introduction

One of the key differences between HTTP APIs and browser‑based traffic is how
errors are communicated to the client. When NGINX is deployed as an API gateway,
we configure it to return errors in a way that best suits the API clients.

For more details read: [Responding to
Errors](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-1/#respond-to-errors)

## Learning Objectives 

By the end of the lab you will be able to: 
 * Configure Custom JSON error pages for API clients

### Configure custom JSON error pages

1. Inspect the NGINX config defining the default MIME type response as
   `application/json` (by default NGINX sets it as `text/plain`):

```bash
$ bat /etc/nginx/nginx.conf
```

2. Lets see a custom JSON error page in action using `curl`:

```bash
$ curl -i https://localhost/foo -k

HTTP/1.1 400 Bad Request
Server: nginx/1.13.10
Date: Fri, 20 Jul 2018 20:14:43 GMT
Content-Type: application/json
Content-Length: 39
Connection: keep-alive

{"status":400,"message":"Bad request"}
```

3. We can see the full list of error responses in the file
   `api_json_errors.conf`

```
$ bat /etc/nginx/api/api_json_errors.conf
```

---------

[Back to Main Menu](../README.md)