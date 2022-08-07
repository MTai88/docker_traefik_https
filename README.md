# Basic `traefik` container setup with https support for local development

### Setup
- create external network `traefik-net`
```
docker network create traefik-net
```

### mkcert
mkcert is a tool to generate locally-trusted development certificates that can be shared with Traefik so it can perform a TLS encryption.
Create and store the TLS certificates with `mkcert` in the `./certs` folder.
These certificates are locally-trusted which mean they are valid only on your machine.
Every developer needs to generate their own certificate when they build the project.

```
mkdir -p certs
mkcert -cert-file certs/local-cert.pem -key-file certs/local-key.pem 
"docker.localhost" "*.docker.localhost" "domain.local" "*.domain.local"
```

### Configure the available subdomains served by Traefik in `traefik/dynamic_conf.yaml`:
```
http:
  routers:
    traefik:
      rule: "Host(`traefik.docker.localhost`)"
      service: "api@internal"
      tls:
        domains:
          - main: "docker.localhost"
            sans:
              - "*.docker.localhost"
          - main: "domain.local"
            sans:
              - "*.domain.local"

tls:
  certificates:
    - certFile: "/etc/certs/local-cert.pem"
      keyFile: "/etc/certs/local-key.pem"
```

### Example of using traefik:
```
version: "3"

services:
  nginx:
    image: nginx:latest
    volumes:
    - ./app/index.html:/app/index.html
    - ./default.conf:/etc/nginx/conf.d/default.conf
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nginx-dev1.rule=Host(`dev1.domain.local`)"
      - "traefik.http.routers.traefik.tls=true"
      - "traefik.http.services.nginx-dev1.loadbalancer.server.port=80"
      - "traefik.docker.network=traefik-net"
    networks:
    - default
    - dev1

  httpd:
    image: httpd:latest
    volumes:
    - ./app/index.html:/usr/local/apache2/htdocs/index.html
    networks:
      - dev1

networks:
  default:
    external: true
    name: traefik-net
  dev1:
    internal: true
```


### Running container
`docker-compose up -d`

### Web interface
`https://traefik.docker.localhost/`