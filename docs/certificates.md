# Certificates

## Introduction

Server certificates are dynamically issued from a Smallstep [Step CA
server](https://smallstep.com/docs/step-ca) running in a docker container. This
container should ideally be brought up before the rest of the routing
infrastructure, so that traefik can immediately start requesting and assigning
certificates to services based on their `Host` rules.

The demo environment showed here, without further configuration, tells `step-ca`
to initialize own root and intermediate CA certificates and keys. However, it
is standard practice to store a root key on an offline host, or hardware
security module (HSM). Further instruction and recommendations for maintaining
an offline root key is outside of the scope of this documentation, although
we'll go through what `step-ca` would need, from this type of environment.

## Existing PKI

If you have your own PKI, then Step can act as an intermeidate, issuing
Certificate Authority. It will need some further manual configuration, however.

### Root CA Certificate

`step-ca` will need a copy of your root CA's public certificate. Copy the root
CA public certificate to `/home/step/certs/root_ca.crt` in your docker
service's volume.

### Create a CSR

`step-ca` will initialise its own root and intermediate keys when initilised
(i.e. when run for the first time).  The public certificates can effectively be
discarded, as we'll use our own root CA certificate, and for the intermediate
certificate, we'll generate a CSR and submit it to the offline root CA for
signing.

The process will look something like this:-

```
# First, start the step-ca service and let it initialise itself:
docker-compose -f docker-compose-step-ca.yaml up

# Copy in your root CA certificate from the host
docker-compose cp certs/root_ca.crt step-ca:/home/step/certs/root_ca.crt

# Create a CSR using the intermediate key generated during initialisation
# First, enter a shell in the container
docker-compose -f docker-compose-step-ca.yaml exec step-ca sh

# Now you're in your container (with its environment variables).
# Generate a CSR from the existing key
step certificate create \
    --csr --password-file="${PWDPATH}" \
    "Example Ltd Intermediate CA" \
    certs/intermediate_ca.csr secrets/intermediate_ca_key

# Exit the container shell, and copy the CSR out and to the offline root CA for
# signing (outside of the scope of this documentation).

exit
docker-compose -f docker-compose-step-ca.yaml \
    cp step-ca:/home/step/certs/intermediate_ca.csr ./certs/

```

### Sign a CSR

Instructions for signing a CSR by a root CA are totally dependent on what CA
software is used by your root CA. Refer to the documentation on your specific
root CA software, for how to sign a CSR for an Intermediate, or Subordinate CA.

### Install Signed Intermediate CA Certificate

```
# Once you've got the signed certificate back, copy it into the container.
docker-compose -f docker-compose-step-ca.yaml \
    cp ./certs/intermediate_ca.crt step-ca:/home/step/certs/

# Restart the Step CA service
docker-compose -f docker-compose-step-ca.yaml restart
```

### Check the certificates

After it is restarted, you can check the public certificate that Step CA is
offering on its public port.

```
# Get the container's IP address from docker:-
IPADDR=$(docker inspect secure-web-env-step-ca-1 | jq  -r '.[].NetworkSettings.Networks.traefik_backend.IPAddress')

# Check the certificate with openssl (assuming listening on default port 9000)
openssl s_client -connect ${IPADDR}:9000
```

## Install Root CA Certificates on traefik
The `traefik` service will handle all incoming traffik. On the backend, it will
also connect to the Step CA service over https, so therefore should trust the
CA certificates used by Step CA.

I chose to build my own traefik image, where the Dockerfile's only
customisation was to `COPY` in the Root and Intermediate CA certificates,
before running `update-ca-certificates`.

The example here doesn't (without further configuraiton) do this. Instead, it
is configured to (insecurely) trust any certificate presented by the CA
backend. See [transports.yaml](../traefik/dynamic-config/transports.yaml),
where `insecureSkipVerify` is set to `true`, and the `rootCAs` block is
commented out. To lock down trust between `traefik` and the Step CA ACME
service, comment out `insecureSkipVerify` and uncomment the `rootCAs` certificates
block, after copying those certificates into the traefik volume.

