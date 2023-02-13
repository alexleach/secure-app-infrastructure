# Traefik Configuration

The `traefik` router has to be configured in a number of ways, as its
configuration is conceptually split between a `static` configuration loaded at
container startup, and a `dynamic` configuration, that can be updated while
`traefik` is running. There are several different dynamic configuration
providers as well, which don't all support the same options. For instance, the
`docker` dynamic configuration provider doesn't support the `serversTransport`
top-level configuration namespace, but the `file` dynamic configuration
provider does. So, `traefik` has to be configured in at least three different
manners:-

- [`traefik.yml`](../traefik/traefik.yml) static configuration file
- [`/var/run/traefik/`](../traefik/dynamic-config/) folder of dynamic
  configuration files
- Environment variables for the LEGO ACME client, defined in
  [`docker-compose-router.yaml`](../docker-compose-router.yaml)
- `docker` labels, on every routed service

`docker` labels are added to traefik itself and every service container which
`traefik` needs to route traffic to, or communicate with directly.

## Customisation

### [`traefik.yml`](../traefik/traefik.yml)

Edit the `certificatesResolvers` section to match your CA of choice. You could
use the public Lets Encrypt CA, or your own ACME CA, if you've configured Step
CA.

### [`docker-compose-router.yaml`](../docker-compose-router.yaml)

Read through the file comments and customise as per your requirements:-

- Root and Intermediate CA Certificates mounted in the `volumes` block.
- Also in the `volumes` block, any default certificate and key, if you don't
  want to use the `traefik` self-signed certificate as a default certificate. I
  would recommend creating a self-signed certificate so malicious actors aren't
  aware that you are using `traefik` as a router. Don't advertise the tools
  you're using!
- In the `labels` block, if you want to expose the `traefik` dashboard,
  configure the `Host` rule for the `dashboard` router. Also, change the
  `basicauth` middleware's user name and password, as per your requirements.
  Once `vault` and `traefik-forward-auth` are configured, you could use
  `traefik-auth` middleware instead.
- Change the `LEGO_CA_` environment variables to match your Step CA
  configuration.

### [`transports.yaml`](../traefik/dynamic-config/transports.yaml)

This file isn't required unless you change the traefik labels on the Step CA
to use an http router. In the current configuration, `traefik` routes raw TCP
traffic to the Step CA container. This works fine, however it means you can't
apply `http` based middleware to the CA, for example the modsecurity CRS
middleware.

To change the router to an `http` based router, so you can apply `http`
middleware, Step CA needs to listen on port 443, whereas its container is
configured to listen on port 9000 by default. And if you do change it to an
`http` router, then `traefik` needs to trust Step CA's presented certificate,
and this is where this dynamic configuration file comes into play.

The `http.serversTransport` block in `transports.yaml` is used to check the
certificate on the backend Step CA service. The `serverName` variable should
match the certificates common name, and the `rootCAs` block defines public
certificates with which to verify the certificate's chain.

So, steps to change the router to an http based one, for Step CA:-

Configure Step CA to listen on port 443
```
# Change the address to listen on port 443 in the Step CA configuration
docker-compose -f docker-compose-ca.yaml exec step-ca \
  sed -i 's/\("address": "\)[:0-9]*"/\1:443"/' config/ca.json
```

Edit the `traefik` labels in
[`docker-compose-ca.yaml`](../docker-compose-ca.yaml).

 - Replace any instances of `traefik.tcp.` with `traefik.http.`
 - Change the loadbalancer's server backend port from `9000` to `443`, so it
   should become:-
```
      - "traefik.http.services.step-ca.loadbalancer.server.port=443"
```
 - Uncomment the following two lines, to use the `serversTransport`
   defined in the dynamic configuration file
``` 
      - "traefik.http.services.step-ca.loadbalancer.server.scheme=https"
      - "traefik.http.services.step-ca.loadbalancer.serverstransport=ca@file"
```

Now, you can restart the Step CA container
```
docker-compose -f docker-compose-ca.yaml restart
```

 - Edit the [`transports.yaml`](../traefik/dyanamic-config/transports.yaml)
   dynamic configuration file. The `serverName` and `rootCAs` need to match the
   certificate's chain presented by the backend service. Or, you can uncomment
  `insecureSkipVerify: true` to allow any backend certificate (not recommended).
