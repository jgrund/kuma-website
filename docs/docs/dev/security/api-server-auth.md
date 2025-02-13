# API Server Authentication

By default, Kuma exposes API Server on [ports](../networking/networking.md) `5681` and `5682` (protected by builtin TLS). This server is used for actions like
* Accessing policies and objects
* Managing policies and objects (on Universal)
* Accessing GUI

Authenticated user can be authorized to execute administrative actions such as
* Managing Kuma Secrets (on Universal)
* Generating Dataplane Tokens
* Generating User Tokens

## User Token

User Token is a JWT token that contains
* The name of the user
* The list of groups that user belongs to
* Expiration date of the token

The User Token is signed by a signing key that is autogenerated on the control plane. The signing key is SHA256 encrypted.

You can check for the signing key:

```sh
kumactl get global-secrets
```
which returns something like:
```
NAME                          AGE
user-token-signing-key-1   36m
```

## Implicit groups

A user can be a part of many groups. On top of that, Kuma adds two groups automatically.
* Every authenticated user is a part of `mesh-system:authenticated`
* Every user that do not provide authentication data is a part of `mesh-system:unauthenticated`.

### Usage

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"
1. Access admin user token to be able to generate other user tokens

In order to generate other user tokens, we need to authenticate as admin. When Kuma starts, it generates admin user token and stores it as a [Global Secret](secrets.md).

Use `kubectl` to extract the admin token
```sh
kubectl get secret admin-user-token -n kuma-system --template={{.data.value}} | base64 -d
```

2. Expose Kuma CP outside a cluster and configure `kumactl` with admin user token

In order to access Kuma CP via kumactl, we need to expose Kuma CP to outside a cluster. We can do this in several ways
a) port-forward port 5681
b) Expose port 5681 and protect it by TLS or just expose 5682 (with builtin TLS) of `kuma-control-plane` service via load balancer.
c) Expose port 5681 of `kuma-control-plane` via `Ingress` (for example Kong Ingress Controller) and protect it by TLS

```sh
kumactl config control-planes add \
  --name my-control-plane \
  --address https://<CONTROL_PLANE_ADDRESS>:5682 \
  --auth-type=tokens \
  --auth-conf token=<GENERATED_TOKEN> \
  --ca-cert-file=/path/to/ca.crt # or --skip-verify if you want to skip CP verification
```

3. Generate user tokens

Now that `kumactl` is configured with admin credentials, we can generate other user tokens.

```sh
kumactl generate user-token \
  --name john \
  --group doe \
  --valid-for 24h
```
:::
::: tab "Universal"
1. Access admin user token to be able to generate other user tokens

In order to generate other user tokens, we need to authenticate as admin. When Kuma starts, it generates admin user token and stores it as a [Global Secret](secrets.md).

Execute the following command on the machine on which the control plane is deployed.
```sh
curl http://localhost:5681/global-secrets/admin-user-token | jq -r .data | base64 -d
```

2. Configure `kumactl` with admin user token
```sh
kumactl config control-planes add \
  --name my-control-plane \
  --address https://<CONTROL_PLANE_ADDRESS>:5682 \
  --auth-type=tokens \
  --auth-conf token=<GENERATED_TOKEN> \
  --ca-cert-file=/path/to/ca.crt # or --skip-verify if you want to skip verifying CP
```

3. Generate user tokens

Now that `kumactl` is configured with admin credentials, we can generate other user tokens.

```sh
kumactl generate user-token \
  --name john \
  --group doe \
  --valid-for 24h
```

4. Disable localhost is admin

By default, all requests that originates from localhost are authenticated as user of name `admin` that belongs to group `mesh-system:admin`.
After you retrieve and store the admin token, it is recommended to [configure a control plane](../documentation/configuration.md) with `KUMA_API_SERVER_AUTHN_LOCALHOST_IS_ADMIN` set to `false`.
:::
::::

### Bootstrap of admin user token

To generate user tokens, we need to first access control plane as admin.
Like we saw in previous section, Kuma creates admin user token when control plane starts.
If you want to remove default admin user token. 

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"
1. Delete `admin-user-token` Secret
```sh
$ kubectl delete secret admin-user-token -n kuma-namespace
```

2. Disable bootstrap of the token
[Configure a control plane](../documentation/configuration.md) with `KUMA_API_SERVER_AUTHN_TOKENS_BOOTSTRAP_ADMIN_TOKEN` set to `false`.
:::
::: tab "Universal"
1. Delete `admin-user-token` Global Secret
```sh
kumactl delete global-secret admin-user-token
```

2. Disable bootstrap of the token
[Configure a control plane](../documentation/configuration.md) with `KUMA_API_SERVER_AUTHN_TOKENS_BOOTSTRAP_ADMIN_TOKEN` set to `false`.
:::
::::

### Token revocation

Kuma does not keep the list of issued tokens. Whenever the single token is compromised, we can add it to revocation list so it's no longer valid.

Every user token has its own ID which is available in payload under `jti` key. You can extract ID from token using jwt.io or [`jwt-cli`](https://www.npmjs.com/package/jwt-cli) tool. Here is example of `jti`
```
0e120ec9-6b42-495d-9758-07b59fe86fb9
```

Specify list of revoked IDs separated by `,` and store it as `GlobalSecret` named `user-token-revocations`

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"
```sh
REVOCATIONS=$(echo '0e120ec9-6b42-495d-9758-07b59fe86fb9' | base64) && echo "apiVersion: v1
kind: Secret
metadata:
  name: user-token-revocations
  namespace: kuma-system 
data:
  value: $REVOCATIONS
type: system.kuma.io/global-secret" | kubectl apply -f -
```
:::
::: tab "Universal"
```sh
echo "
type: GlobalSecret
name: user-token-revocations
data: {{ revocations }}" | kumactl apply --var revocations=$(echo '0e120ec9-6b42-495d-9758-07b59fe86fb9' | base64) -f -
```
:::
::::

### Signing key rotation

If the signing key is compromised, we must rotate it and all the tokens that was signed by it.

1. Generate new signing key
   The signing key is stored as a `GlobalSecret` with a name that looks like `user-token-signing-key-{serialNumber}`.

   Make sure to generate the new signing key with a serial number greater than the serial number of the current signing key.

   :::: tabs :options="{ useUrlFragment: false }"
   ::: tab "Kubernetes"
   Check what is the current highest serial number.
   
   ```sh
   kubectl get secrets -n kuma-system --field-selector='type=system.kuma.io/global-secret'
   NAME                          TYPE                           DATA   AGE
   user-token-signing-key-1   system.kuma.io/global-secret   1      25m
   ```
   
   In this case, the highest serial number is `1`. Generate a new signing key with a serial number of `2`
   ```sh
   TOKEN="$(kumactl generate signing-key)" && echo "
   apiVersion: v1
   data:
     value: $TOKEN
   kind: Secret
   metadata:
     name: user-token-signing-key-2
     namespace: kuma-system
   type: system.kuma.io/global-secret
   " | kubectl apply -f - 
   ```
   
   :::
   ::: tab "Universal"
   Check what is the current highest serial number.
   ```sh
   kumactl get global-secrets
   NAME                             AGE
   user-token-signing-key-1   36m
   ```
   
   In this case, the highest serial number is `1`. Generate a new signing key with a serial number of `2`
   ```sh
   echo "
   type: GlobalSecret
   name: user-token-signing-key-2
   data: {{ key }}" | kumactl apply --var key=$(kumactl generate signing-key) -f -
   ```
   :::
   ::::

2. Regenerate user tokens
   Create new user tokens. These tokens are automatically created with the signing key that’s assigned the highest serial number, so they’re created with the new signing key.
   At this point, tokens signed by either new or old signing key are valid.

3. Remove the old signing key
   :::: tabs :options="{ useUrlFragment: false }"
   ::: tab "Kubernetes"
   ```sh
   kubectl delete secret user-token-signing-key-1 -n kuma-system
   ```
   :::
   ::: tab "Universal"
   ```sh
   kumactl delete global-secret user-token-signing-key-1
   ```
   :::
   ::::
   All new connections to the control plane now require tokens signed with the new signing key.

### Explore an example token

You can decode the tokens to validate the signature or explore details.

For example, run:

```sh
kumactl generate user-token --name=john --group=team-a --valid-for=24h
```

which returns:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6IjEiLCJ0eXAiOiJKV1QifQ.eyJOYW1lIjoiam9obiIsIkdyb3VwcyI6WyJ0ZWFtLWEiXSwiZXhwIjoxNjM2ODExNjc0LCJuYmYiOjE2MzY3MjQ5NzQsImlhdCI6MTYzNjcyNTI3NCwianRpIjoiYmYzZDBiMmUtZDg0MC00Y2I2LWJmN2MtYjkwZjU0MzkxNDY4In0.XsaPcQ5wVzRLs4o1FWywf6kw4r2ceyLGxYO8EbyA0fAxU6BPPRsW71ueD8ZlS4JlD4UrVtQQ7LG-z_nIxlDRAYhx4mmHnSjtqWZIsVS13QRrm41zccZ0SKHYxGvWMW4IkGwUbA0UZOJGno8vbpI6jTGfY9bmof5FpJJAj_sf99jCaI1H_n3n5UxtwKVN7dXXD82r6axj700jgQD-2O8gnejzlTjZkBpPF_lGnlBbd39S34VNwT0UlvRJLmCRdfh5EL24dFt0tyzQqDG2gE1RuGvTV9LOT77ZsjfMP9CITICivF6Z7uqvlOYal10jd5gN0A6w6KSI8CCaDLmVgUHvAw
```

Paste the token into the UI at jwt.io, or use [`jwt-cli`](https://www.npmjs.com/package/jwt-cli) tool

```sh
kumactl generate user-token --name=john --group=team-a --valid-for=24h | jwt

To verify on jwt.io:

https://jwt.io/#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjEiLCJ0eXAiOiJKV1QifQ.eyJOYW1lIjoiam9obiIsIkdyb3VwcyI6WyJ0ZWFtLWEiXSwiZXhwIjoxNjM2ODExNjc0LCJuYmYiOjE2MzY3MjQ5NzQsImlhdCI6MTYzNjcyNTI3NCwianRpIjoiYmYzZDBiMmUtZDg0MC00Y2I2LWJmN2MtYjkwZjU0MzkxNDY4In0.XsaPcQ5wVzRLs4o1FWywf6kw4r2ceyLGxYO8EbyA0fAxU6BPPRsW71ueD8ZlS4JlD4UrVtQQ7LG-z_nIxlDRAYhx4mmHnSjtqWZIsVS13QRrm41zccZ0SKHYxGvWMW4IkGwUbA0UZOJGno8vbpI6jTGfY9bmof5FpJJAj_sf99jCaI1H_n3n5UxtwKVN7dXXD82r6axj700jgQD-2O8gnejzlTjZkBpPF_lGnlBbd39S34VNwT0UlvRJLmCRdfh5EL24dFt0tyzQqDG2gE1RuGvTV9LOT77ZsjfMP9CITICivF6Z7uqvlOYal10jd5gN0A6w6KSI8CCaDLmVgUHvAw

✻ Header
{
  "alg": "RS256",
  "kid": "1",
  "typ": "JWT"
}

✻ Payload
{
  "Name": "john",
  "Groups": [
    "team-a"
  ],
  "exp": 1636811674,
  "nbf": 1636724974,
  "iat": 1636725274,
  "jti": "bf3d0b2e-d840-4cb6-bf7c-b90f54391468"
}
   Issued At: 1636725274 11/12/2021, 2:54:34 PM
   Not Before: 1636724974 11/12/2021, 2:49:34 PM
   Expiration Time: 1636811674 11/13/2021, 2:54:34 PM

✻ Signature XsaPcQ5wVzRLs4o1FWywf6kw4r2ceyLGxYO8EbyA0fAxU6BPPRsW71ueD8ZlS4JlD4UrVtQQ7LG-z_nIxlDRAYhx4mmHnSjtqWZIsVS13QRrm41zccZ0SKHYxGvWMW4IkGwUbA0UZOJGno8vbpI6jTGfY9bmof5FpJJAj_sf99jCaI1H_n3n5UxtwKVN7dXXD82r6axj700jgQD-2O8gnejzlTjZkBpPF_lGnlBbd39S34VNwT0UlvRJLmCRdfh5EL24dFt0tyzQqDG2gE1RuGvTV9LOT77ZsjfMP9CITICivF6Z7uqvlOYal10jd5gN0A6w6KSI8CCaDLmVgUHvAw
```

## Admin Client Certificates

This section describes alternative way of authenticating to API Server.
Admin client certificates are deprecated. If you are using it, please migrate to user token described above. This will be removed in Kuma 1.5.0.

To use admin client certificates, set `KUMA_API_SERVER_AUTHN_TYPE` to `adminClientCerts`.

All users that provides client certificate are authenticated as user with name `admin` that belongs to group `admin`. 

### Usage

1. Generate client certificates by using kumactl
   ```sh
   kumactl generate tls-certificate --type=client \
     --cert-file=/tmp/tls.crt \
     --key-file=/tmp/tls.key
   ```

2. Configure the control plane with client certificates
   :::: tabs :options="{ useUrlFragment: false }"
   ::: tab "Kubernetes (kumactl)"
   Create a secret in the namespace in which control plane is installed
   ```sh
   kubectl create secret generic api-server-client-certs -n kuma-system \
     --from-file=client1.pem=/tmp/tls.crt \
   ```
   We can provide as many client certificates as we want. Remember to only provide certificates without keys.
   
   Point to this secret when installing Kuma
   ```sh
   kumactl install control-plane \
     --tls-api-server-client-certs-secret=api-server-client-certs
   ```
   :::
   ::: tab "Kubernetes (HELM)"
   Create a secret in the namespace in which control plane is installed
   ```sh
   kubectl create secret generic api-server-client-certs -n kuma-system \
     --from-file=client1.pem=/tmp/tls.crt \
   ```
   We can provide as many client certificates as we want. Remember to only provide certificates without keys.
   
   Set `controlPlane.tls.apiServer.clientCertsSecretName` to `api-server-client-certs` via HELM
   :::
   ::: tab "Universal"
   Put all the certificates in one directory
   ```sh
   mkdir /opt/client-certs
   cp /tmp/tls.crt /opt/client-certs/client1.pem 
   ```
   All client certificates **MUST** end with `.pem` extension. Remember to only put provide certificates without keys.
   
   Configure control plane by pointing to this directory
   ```sh
   KUMA_API_SERVER_AUTH_CLIENT_CERTS_DIR=/opt/client-certs \
     kuma-cp run
   ```
   :::
   ::::

3. Configure `kumactl` with valid client certificates
   ```sh
   kumactl config control-planes add \
     --name=<NAME>
     --address=https://<KUMA_CP_DNS_NAME>:5682 \
     --client-cert-file=/tmp/tls.crt \
     --client-key-file=/tmp/tls.key \
     --ca-cert-file=/tmp/ca.crt # CA cert used in "Encrypted communication" section
   ```

## Multizone

In multizone setup, the majority of actions are executed on the global control plane.
However, some actions like generating dataplane tokens are available on zone control planes.
Authentication credentials are not propagated from global control plane to zone control planes.
Consistent user tokens across the whole setup can be achieved by manually synchronizing signing key from global to zone control planes. 
