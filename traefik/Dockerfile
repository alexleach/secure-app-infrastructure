FROM traefik:latest

# Add the private CA certificates into the custom CA certificate store folder
COPY certs/*.crt /usr/local/share/ca-certificates/

# Add the CA certificates to the system store
RUN update-ca-certificates
