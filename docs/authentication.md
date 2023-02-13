# Introduction

As part of our infrastructure, we can configure a single sign-on provider, so
users trying to access services can login just once, and be authenticated to
access all services for which they are authorised.

The [docker-compose-auth.yaml](../docker-compose-auth.yaml) file is configured
to run a Vault OpenID Connect (OIDC) Service, using Consul as backend storage.
Vault's OIDC provider can be used as authentication provider to a traefik
middleware,
[traefik-forward-auth](https://github.com/thomseddon/traefik-forward-auth), or
for any OIDC capable service.

The workflow is as follows:-

  1.  User accesses https://service.example.com
  2.  HTTP Headers are checked to see if user is authenticated for example.com
  3.  If none exist, or the access tokens are expired, user is redirected to
      https://vault.example.com/ where they can login
  4.  Upon successful login, the user is issued Access Tokens for example.com

# Setup

The main component configured here is an authentication server, running
Hashicorp's [Vault](https://www.vaultproject.io/) server. There are two
additional components which Vault depends on: an Identity Provider (IdP), and a
storage backend.

In this example configuration, Vault is configured to use Consul as a storage
backend. Consul is designed as a highly-available key-value database, so can
easily be scaled to two or more nodes. On a single node, a file storage
provider could be used instead, but that cannot be scaled, so we may as well
start off using a storage backend that can be scaled later.

## Consul

Consul is quite simple to configure for a single instance. The only two
places to configure are the
[docker-compose-auth.yaml](../docker-compose-auth.yaml) file and consul's
static configuration file, [consul-config.json](./consul-config.json).


## Vault OIDC Provider

Vault also uses two configuration files. One, being
[docker-compose-auth.yaml](../docker-compose-auth.yaml) again and the other
being [vault.json](../vault/config/vault.json).

Note that Vault can use either an external Identity Provider (IdP), or can be
configured as its own IdP. The documentation below describes how to set up
Vault as its own IdP, using the `userpass` authentication method.

The IdP is a store of User and Passwords, and will also contain metadata about
users. Other IdPs include cloud-based ones like Azure AD, Google, etc., or
on-premise IdPs like LDAP or RADIUS. See
[here](https://developer.hashicorp.com/vault/docs/concepts/identity#mount-bound-aliases)
for a full list of supported Vault Identity Providers.

Some background reading on Vault OIDC providers:-

  - https://developer.hashicorp.com/vault/docs/concepts/oidc-provider

### Initialising Vault

Vault needs quite a lot of hands-on work to set up, and then also needs manual
intervention every time it is restarted. Further, any new web apps added as
OIDC clients will also need to be manually added to Vault, and configuration
copied back to the OIDC-capable app.

The tutorial on the Vault website should be the main reference point for
configuring Vault as an OIDC identity provider. Please have a read through:-
https://developer.hashicorp.com/vault/tutorials/auth-methods/oidc-identity-provider

However, the steps followed are noted down here:-

```
# Set an alias for the `vault` command
alias vault='docker-compose -f docker-compose-auth.yaml exec vault vault "$@"'

# Initialise vault. Add --help flag to see all available options
vault operator init
```

Take a note of the unseal keys and root token. There are various security
related best-practifces about these keys and token, e.g. encrypt the unseal
keys with gpg keys, distribute amongst numerous people, disable the root token
account, etc.

### Unsealing Vault and logging in

Unseal vault and login with the root token (or a user role with the privileges 
defined in the policy at the top of the Vault oidc-identity-provider
documentation)
```
vault operator unseal
vault login
```

### Configure an Identity Provider.

For the purposes of this setup, we can enable Vault's internal `userpass` IdP.
(See [Vault's docs](https://developer.hashicorp.com/vault/docs/auth) for a full
list of Auth Methods built into Vault.)

```
# Enable the userpass Auth method
vault auth enable userpass
```

Let's create a user, say `j.bloggs` and set an initial password

```
vault write auth/userpass/users/j.bloggs password='foo123' token_ttl="1h"
```

### Create a vault Entity and Group

This links the user entity in Vault to the backend IdP and adds some metadata.

```
vault write identity/entity name="j.bloggs" \
    metadata="email=j.bloggs@example.com" \
    metadata="phone_number=+1234567890" \
    disabled=false
```

Create a group, so policies can be applied to multiple user entities.

```
vault write identity/group name="staff" member_entity_ids="${IDENTITY_ID}"
```

Get Unique IDs for the `userpass` auth method, the user Entity, and the Group.

```
USERPASS_ACCESSOR=$(vault auth list -detailed -format json | jq -r '.["userpass/"].accessor')
ENTITY_ID=$(vault read -field=id identity/entity/name/j.bloggs)
GROUP_ID=$(vault read -field=id identity/group/name/all_users)
```

Create an `entity-alias`, which associates the Auth Method (i.e. `userpass`
backend) with the relevant User Name for that backend. Multiple alias's can be
configured, but each alias needs to be associated with a different Auth Method
(the `mount_accessor` argument below).

```
vault write identity/entity-alias name="j.bloggs" \
    canonical_id="${ENTITY_ID}" \
    mount_accessor="${USERPASS_ACCESSOR}"
```

### Configure the OIDC provider

Create an 'assignment', which is a grouping of `entity_ids` and `group_ids`
that can be assigned as like an Access Control Entry.

```
vault write identity/oidc/assignment/allowed_users \
    entity_ids="${ENTITY_ID}" \
    group_ids="${GROUP_ID}"
```

Create a key to be used by the OIDC provider.

```
vault write identity/oidc/key/oidc-key \
   allowed_client_ids="*" \
   verification_ttl="2h" \
   rotation_period="1h" \
   algorithm="RS256"
```

### Add an OIDC client application

Create your first OIDC client, for `traefik-forward-auth`. The `redirect_uris`
here will need to match the `AUTH_HOST` and `URL_PATH` environment variables
defined for the `forward-auth` service, configured in
[`docker-compose-auth.yaml`](../docker-compose-auth.yaml)

```
vault write identity/oidc/client/traefik \
    redirect_uris="https://auth.example.com/auth/oidc-callback" \
    assignments="allowed_users" \
    key="oidc-key" \
    id_token_ttl="30m" \
    access_token_ttl="1h"
```

Now, copy across the `CLIENT_ID` and `CLIENT_PASSWORD` values into the
respective `PROVIDERS_OIDC_` environment variables in 
[`docker-compose-auth.yaml`](../docker-compose-auth.yaml).

Further, save the client ID to a bash variable.

```
CLIENT_ID=$(vault read -field=client_id identity/oidc/client/traefik)
```

### Configure the Vault OIDC provider

We specify what attributes (claims) can be requested in scope templates.

```
USER_SCOPE_TEMPLATE='{
    "username": {{identity.entity.name}},
    "contact": {
        "email": {{identity.entity.metadata.email}},
        "phone_number": {{identity.entity.metadata.phone_number}}
    }
}'

GROUPS_SCOPE_TEMPLATE='{
    "groups": {{identity.entity.groups.names}}
}'

EMAIL_SCOPE_TEMPLATE='{
    "email": {{identity.entity.metadata.email}}
}'

# Add the user scope template to vault's database
vault write identity/oidc/scope/user \
    description="The user scope provides claims using Vault identity entity metadata" \
    template="$(echo ${USER_SCOPE_TEMPLATE} | base64 -)"

# Add the groups scope template to vault's database
vault write identity/oidc/scope/groups \
    description="The groups scope provides the groups claim using Vault group membership" \
    template="$(echo ${GROUPS_SCOPE_TEMPLATE} | base64 -)"

# Add an email scope template to vault's database
vault write identity/oidc/scope/email \
    description="The email scope creates a top level 'email' claim" \
    template="$(echo ${EMAIL_SCOPE_TEMPLATE} | base64 -)"
```

### Configure the Vault provider

Finally, create the example.com provider, adding the `traefik` OIDC client
to the list of allowed clients, and the scopes that are supported

```
# Configure the scopes and clients supported, by creating a 'provider'
vault write identity/oidc/provider/example.com \
    allowed_client_ids="${CLIENT_ID}" \
    scopes_supported="groups,user,email"
```

Now, you should be able to check the OIDC configuration endpoint!
```
curl -s https://vault.example.com/v1/identity/oidc/provider/example.com/.well-known/openid-configuration | jq
```

Some notes on the Vault website's
[tutorial](https://developer.hashicorp.com/vault/tutorials/auth-methods/oidc-identity-provider):-

  - No need to worry about writing a custom vault 'policy' on a new server. The
    default policy works with the OpenID Connect provider out of the box
    (tested vault v1.12.2), and the root token can be used to set everything
    up. It is recommended to set up a user with appropriate permissions though,
    instead of using the root token.
  - Each user entity needs an alias configured with the same name. This first
    alias is required to point the entity to a specific `mount_accessor`.
  - Each additional alias requires a different `mount_accessor`. The
    `mount_accessor` is effectively the identity provider. In this case, we use
    the `userpass` authentication module.  So, you can't configure two aliases
    that login with the same password and backend. I tried configuring my email
    as an alias, so I could login either with my username or my email, both using
    the same backend entity and password. This didn't work :/
  - The key needs to be configured with the `RS256` algorithm. For whatever
    reason, `traefik-forward-auth` doesn't accept for example ES384 keys.
  - `email` needs to be added as a scope's top level key. I created a separate
    `email` scope for this (but it could probably just be moved out of the
    `contact` group in the `user` scope):-


## traefik-forward-auth

Now that Vault is configured as an authentication provider, we can start
configuring client apps to authenticate users with it. Not all web apps support
OIDC authentication out of the box (e.g. Jupyter Notebooks, without a Jupyter
Hub instance), but we can configure `traefik` middleware to only allow
authenticated users to access any particular service.  We will do this with the
excellent
[`traefik-forward-auth`](https://github.com/thomseddon/traefik-forward-auth)
traefik plugin.

If you've followed the steps above, you'll already have [created an OIDC
client](#add-an-oidc-client-application) in Vault.

Once you've regsistered the OIDC client app in Vault, get its `CLIENT_ID` and
`CLIENT_SECRET` using:-

```
vault read identity/oidc/client/traefik-auth
```

The `CLIENT_ID` and `CLIENT_SECRET` need to be added to the
`traefik-forward-auth` service in
[docker-compose-auth.yaml](../docker-compose-auth.yaml)

Once this is configured (and the service restarted of course!), you can start
protecting traefik services with it, by adding docker labels to each service
you want to protect.

```
    labels:
      [...]
      - "traefik.http.routers.<router>.middlewares=traefik-forward-auth"
```

A fully protected example web service running the `whoami` image is
demonstrated in [`docker-compose-whoami.yaml`](../docker-compose-whoami.yaml).

### Enable TOTP MFA

As a bonus, we can further protect the web services by enabling Time-based
One-Time Password (TOTP) Multi-Factor Authentication in Vault.

Used the documentation at
https://developer.hashicorp.com/vault/tutorials/auth-methods/active-directory-mfa-login-totp
but had to amend quite significantly, to work with the `userpass` auth method
and to allow user's to register their own MFA method.

Update the default policy, allowing entity's to generate their own MFA QR code.

`./oidc-data/vault/config/generate-mfa.hcl`
```
# Allow a user to generate their own TOTP QR code and method
path "identity/mfa/method/totp/generate" {
  capabilities = ["create", "update"]
}

# Allow a user to list available TOTP methods
path "identity/mfa/method/totp" {
  capabilities = ["read"]
}
```

Get the current, default policy, and append the above.

```
# Save the file path's to variables
DEFAULT_CONFIG="./vault/policies/default.hcl"
APPEND_MFA_CONFIG="./vault/policies/generate-mfa.hcl"

# Get the current 'default' policy
vault policy read default > "${DEFAULT_CONFIG}"

# Append the policies above to the default policy
cat "${APPEND_MFA_CONFIG}" >> "${DEFAULT_CONFIG}"

# Load the updated policy back into vault
vault policy write default "${DEFAULT_CONFIG}"
```
Now the default user permissions (should) allow entity's to register their own
MFA method!

But, we still need to enable the TOTP auth method...

```
# Create TOTP Auth method and store its method_id
TOTP_METHOD_ID=$(vault write identity/mfa/method/totp \
    -format=json \
    issuer=vault.example.com \
    period=30 \
    key_size=30 \
    algorithm=SHA256 \
    digits=6 | jq -r '.data.method_id')
```

Now, we can link the mfa authentication method to the userpass authenticator.
We need the `TOTP_METHOD_ID` obtained above and the userpass accessor ID,
obtained with:-

`USERPASS_ACCESSOR_ID=$(vault auth list -format=json | jq -r '."userpass/".accessor'`)

Enforce MFA on the userpass accessor:-

```
vault write identity/mfa/login-enforcement/enforce-mfa \
    mfa_method_ids=${TOTP_METHOD_ID} \
    auth_method_accessors=${USERPASS_ACCESSOR_ID}
```

### Generate and import QR Code

The final step is generating a QR code for an end user / entity. Unfortunately,
the web ui doesn't (yet?) support generating and showing end user QR codes, so
we have to do it on the command line. The web UI does allow users's to run
these commands in the browser though.

The QR code can be scanned by an authenticator app on a smart phone or tablet.

```
# Login as user entity, and save its entity_id
ENTITY_ID=$(vault login -method=userpass username=$(whoami) -format=json | jq -r '.auth.entity_id')

# Create a QR code for the local user
vault write identity/mfa/method/totp/generate method_id="${TOTP_METHOD_ID}"
```

The previous `vault write` command returns two items: `barcode`, and `url`.
The `barcode` is a base64-encoded png image. It should be converted to png
format:-

```
QR_CODE="<copy and paste the 'barcode' value here>"
echo "${QR_CODE}" | base64 --decode - > my-totp-qr-code.png
```

Open the png file in an image viewer, and scan it with your authenticator app,
which will start generating TOTP codes.


