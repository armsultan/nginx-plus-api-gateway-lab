# NGINX Plus API Gateway Demo/Workshop

This demo package is based on the techical blog post **Deploying NGINX Plus as
an API Gateway** ([Part
1](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-1/),
[Part
2](https://www.nginx.com/blog/deploying-nginx-plus-as-an-api-gateway-part-2-protecting-backend-services/))

## The Demo environment

This demo has two components, an NGINX Plus API Gateway
(`nginx-plus-api-gateway`) and "API services" (`nginx1` and `nginx2`):

 * **NGINX Plus** `nginx-plus-api-gateway` on Ubuntu. [NGINX Plus
   Documentation](https://docs.nginx.com/nginx/) and
   [resources](https://www.nginx.com/resources/) and
   [blog](https://www.nginx.com/blog/) is your best source of information for
   technical help. Detailed examples are found on the internet too!

 * "**API services**" (`nginx1` and `nginx2`) are NGINX webservers on Debian
   Linux that serves a simple page API endpoint, responding with its hostname,
   IP address and port, and the request URI and the local time of the webserver

 * **NGINX Instance Manager** (`nginx-instance-manager`) on Ubuntu. NIM track
   all NGINX Open Source and NGINX Plus instances. NGINX Agents are installed
   all NGINX instances in this environement to allow to you inspect and manage
   the NGINX confifurations

## File Structure

Centered in the demo environment is the NGINX Plus API Gateway Configurations

We create a configuration layout that supports a multi‑purpose NGINX Plus
deployment and provides a convenient structure for automating configuration
deployment through CI/CD pipelines to achieve this separation. The resulting
directory structure under `/etc/nginx` looks like this:

```
etc/
├── nginx/
│    ├── api/ …………………………………………………………………… Subdirectory for Global API configurations
│    |   └── api_backends.conf ………………………………………… The backend services (upstreams)
│    |   └── api_keys.conf …………………………………………………… API keys
│    |   └── api_gateway.conf …………………………………………… Top-level configuration for the API gateway server
│    |   └── api_json_errors.conf ………………………………… HTTP error responses in JSON format
│    |   └── api_jwt.conf ……………………………………………………… Global settings for JWT validation
│    ├── api_conf.d/ ………………………………………………… Subdirectory for per-API configuration
│    │   └── warehouse_api.conf …………………………… Definition and policy of the Warehouse API
│    │   └── warehouse_api_apikeys.conf ……… API key authenication
│    │   └── warehouse_api_bodysize.conf …… Enforce client request body size
│    │   └── warehouse_api_jwt.conf ………………… JWT Validation
│    │   └── warehouse_api_methods.conf ……… Enforce HTTP Methods
│    │   └── warehouse_api_precise.conf ……… Precise API endpoint definitions
│    │   └── warehouse_api_rewrites.conf …… Rewrite rules for API endpoints
│    │   └── warehouse_api_simple.conf ………… Broad and simple API endpoint definitions
│    ├── conf.d/……………………………………………………………… Subdirectory for other HTTP configurations (Web │server, load balancing, etc.)
│    │   └── demo_instructions.conf ………………… Demo Instructions on port 9000
│    │   └── echo_json.conf ……………………………………… Dummy API Servers - static json responses
│    │   └── health_checks.conf …………………………… Example active health check probes
│    │   └── status_api.conf …………………………………… NGINX Plus live activity monitoring API on port │8080
│    │   └── status_stub.conf ………………………………… NGINX OSS simple live metrics
│    ├── js/………………………………………………………………………… Subdirectory for NGINX JavaScript files
│    |   └── json_validator.js ……………………………… NGINX javascript for json validation
│    ├── jwk/……………………………………………………………………… JSON Web Keys used to validate JWT
│    ├── jwt/……………………………………………………………………… JSON Web Tokens used to demo JWT validation
│    └── nginx.conf …………………………………………………………… Main NGINX configuration file
└── ssl/
    ├── example.com/ ………………………………………………… Self signed cert for HTTPS testing
    │   └── api.example.com.crt …………………………… Self signed cert for api.example.com
    │   └── api.example.com.key …………………………… Private key for api.example.com
    ├── jwt_key_cert/ ……………………………………………… Self signed certs used to generate the JWT and JWK
    └── nginx/ ………………………………………………………………… NGINX Plus repo.crt and repo.key goes here
```

The directories and filenames for all API gateway configuration are prefixed
with api_. Each of these files and directories enables different features and
capabilities of the API gateway and illustrated in the demos

### Topology

The base demo environment you are tasked to build from

```
                                        (both upstream listen on ports:
                 +---------------+       8811, 8822, 8833, 8844)                 
                 |               |       +-----------------+
                 |               +------>|                 |
                 |               +------>|      nginx1     |
                 |               +------>|("Warehouse API")|
                 |               +------>|                 |
+---------------->               |       +-----------------+
www.example.com  |               |
HTTPS/Port 80/443|               |       +-----------------+
                 |  nginx-plus   +------>|                 |
                 | (API Gateway) +------>|      nginx2     |
                 |               +------>|("Inventory API")|
                 |               c------>|                 |
                 |               |       +-----------------+
+---------------->               |                 
NGINX Dashboard/ |               |                 
API              |               |                 
HTTP/Port 8080   |               |                               
                 |               |                 
                 +---------------+                     
                                                                                           
                 +---------------+                 
                 |               |                 
+---------------->     NGINX     +
NIM UI/API       |    Instance   |
HTTP/PORT 9080   |    Manager    | 
                 +---------------+                               
```

## Prerequisites:

1. NGINX Instance Manager evaluation license files (Includes NGINX Plus
   evaluation ). You can get it from you MyF5 Portal or ask your NGINX
   Specialist for one!

2. A Docker host. With [Docker](https://docs.docker.com/get-docker/) and [Docker
   Compose](https://docs.docker.com/compose/install/)

3. **Optional**: The demo uses the hostname `api.example.com` . For hostname
   resolution, you will need to add hostname bindings to your hosts file:

For example, on Linux/Unix/macOS, the host file is `/etc/hosts`

```bash
# NGINX Plus API Gateway demo (local docker host)
127.0.0.1 api.example.com 
```

> **Note:** DNS resolution between containers is provided by default using a new
> bridged network by docker networking and NGINX has been preconfigured to use
> the Docker DNS server (127.0.0.11) to provide DNS resolution between NGINX and
> upstream servers


### Just add Nginx Plus License file

In order to build the Docker compose stack and run Nginx Plus with NGINX
Instance Manager you need to copy your Nginx Plus license or Evaluation Trial
License.

To simplify the this requirement, you can can simply use the NGINX Instance
Manager Trial license (**Ask your NGINX Specialist for one!**)

Place the following files in the expected directories:
 * **For NGINX Plus and NGINX Agent**: `nginx-repo.key` and `nginx-repo.crt`)
   into the `nginx-plus/etc/ssl/nginx/`  
 * **For NGINX Agent**:`nginx-repo.key` and `nginx-repo.crt`) into the
   `nginx/etc/ssl/nginx/`  
 * **For NGINX Instance Manager, NGINX Plus and NGINX Agent**`nginx-repo.key`
   and `nginx-repo.crt`) into the `nginx-instance-manager/etc/ssl/nginx/`, and
   `nginx-manager.lic` into `nginx-instance-manager/etc/nginx-manager/`


## Build and run the demo environment

Provided the Prerequisites have been met before running the steps below, this is
a **working** environment. 

### Build the demo

In this demo, we will have a one NGINX Plus API Gateway (`nginx-plus`) and two
NGINX OSS webserver (`nginx1` and `nginx2`). We Will also have one NGINX
Instance Manager as an option to experience managing NGINX Plus and OSS
instances

Before we can start, we need to copy our NGINX Plus repo key and certificate
(`nginx-repo.key` and `nginx-repo.crt`) into the directory,
`nginx-plus/etc/ssl/nginx/`, then build our stack:

```bash
# Enter working directory
$ cd nginx-plus-api-gateway-nim-demo

# Make sure your Nginx Plus repo key and certificate exist in these folders
$ ls nginx-plus/etc/ssl/nginx/nginx-*

nginx-repo.crt              nginx-repo.key
# here
$ ls nginx/etc/ssl/nginx/nginx-*

nginx-repo.crt              nginx-repo.key
# and here
$ ls nginx-instance-manager/etc/ssl/nginx/nginx-*

nginx-repo.crt              nginx-repo.key

# We also need the NGINX Instance Manager license file here
$ ls nginx-instance-manager/etc/nginx-manager/nginx-manager.lic 

nginx-instance-manager/etc/nginx-manager/nginx-manager.lic

# Downloaded docker images and build
$ docker-compose pull
$ docker-compose build --no-cache
```

#### Start the Demo stack:

Run `docker-compose` in the foreground so we can see real-time log output to the
terminal:

```bash
$ docker-compose up -d
```

Or, if you made changes to any of the Docker containers or NGINX configurations,
run:

```bash
# Recreate containers and start demo
$ docker-compose up --force-recreate -d
```

**Confirm** the containers are running. You should see three containers running:

```
$ docker ps

CONTAINER ID   IMAGE                                       COMMAND            CREATED              STATUS              PORTS                                                                                                                                                                                           NAMES
a47feb37959e   nginx-api-gateway-nim-demo-nginx-plus-api-gateway   "/entrypoint.sh"   56 seconds ago       Up 55 seconds       0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:9000->9000/tcp, :::9000->9000/tcp                                  nginx-plus-api-gateway
13c39c2b88fe   nginx-api-gateway-nim-demo-nginx2                   "/entrypoint.sh"   56 seconds ago       Up 55 seconds       80/tcp, 443/tcp, 8811/tcp, 8822/tcp, 8833/tcp, 8844/tcp                                                                                                                                         nginx-api-gateway-nim-demo-nginx2_1
d661be4a640c   nginx-api-gateway-nim-demo-nginx-instance-manager   "/entrypoint.sh"   56 seconds ago       Up 55 seconds       0.0.0.0:9090->9090/tcp, :::9090->9090/tcp, 8080/tcp, 11000/tcp, 0.0.0.0:10443->10443/tcp, :::10443->10443/tcp, 0.0.0.0:9080->80/tcp, :::9080->80/tcp, 0.0.0.0:9443->443/tcp, :::9443->443/tcp   nginx-api-gateway-nim-demo-nginx-instance-manager_1
83dc966c986e   nginx-api-gateway-nim-demo-nginx1                   "/entrypoint.sh"   About a minute ago   Up About a minute   80/tcp, 443/tcp, 8811/tcp, 8822/tcp, 8833/tcp, 8844/tcp                                                                                                                                         nginx-api-gateway-nim-demo-nginx1_1
```

## Demo Instructions:

1. Open the Lab guide in a Web browser on
   [http://localhost:9000](http://localhost:9000) and follow the getting started
   guide

![Table Of Contents](/labs/setup/media/table-of-contents.png)

## Troubleshooting

See other other useful [`docker`](docs/useful-docker-commands.md) and
[`docker-compose`](docs/useful-docker-compose-commands.md) commands

