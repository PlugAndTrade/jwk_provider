# JwkProvider

Stores and serves certificates, in jwk format, read from file system or provided by Vault.

## Installation

```elixir
def deps do
  [
    {:jwk_provider, github: "PlugAndTrade/jwk_provider"}
  ]
end
```

## Configuration

```elixir
config :jwk_provider, :provider,
  {:system, :atom, "JWK_PROVIDER", :fs}

config :jwk_provider, JwkProvider.FileSystem,
  public_key: {:system, "FS_PUBLIC_KEY", "./priv/certs/trusted_service.crt"},
  private_key: {:system, "FS_PRIVATE_KEY", "./priv/certs/trusted_service.key"}

config :jwk_provider, JwkProvider.Vault,
  url: {:system, "VAULT_URL", "https://localhost:8200"},
  ca_fingerprint: {:system, "VAULT_CA_FINGERPRINT"},
  token: {:system, "VAULT_TOKEN", "myroot"},
  pki_path: {:system, "VAULT_PKI_PATH", "jwt_ca"},
  pki_role: {:system, "VAULT_PKI_ROLE", "trusted_service"},
  common_name: {:system, "VAULT_PKI_CN", "trusted_service"},
  expire_margin: {:system, "VAULT_EXPIRE_MARGIN", 60}
```

### Setup fs

Set `JWK_PROVIDER` to `fs`.

Public/private key pair may be read from files, one private key and one public.
These can be generated with `mix certs.generate`, which will generate a new
pair at `priv/certs/rsa_test_dev.{pem,pub}`.

Certificate, `priv/certs/trusted_service.crt`, with private key,
`priv/certs/trusted_service.key`, may be generated with
```
docker run --rm \
  -v $PWD/priv/certs:/var/tls \
  -e CERT_PATH=/var/tls \
  -e COMMON_NAME=trusted_service \
  -e SUBJECT='/C=SE/O=Dev' \
  -e VALID_DAYS=365 \
  plugandtrade/certificate-generator:1.0.0
```

Set `FS_PUBLIC_KEY` and `FS_PRIVATE_KEY` to the paths of your keys. They
default to `priv/certs/trusted_service.{crt,key}` for convenience during
development.

### Setup Vault

Set `JWK_PROVIDER` to `vault`.

Certificates may be generated by vault using the PKI bachend,
https://www.vaultproject.io/docs/secrets/pki/index.html.

Vault specific configuration:
  * `VAULT_URL` The url to vault, must be `https`, default: `https://localhost:8200`.
  * `VAULT_CA_FINGERPRINT` The `sha256` fingerprint of https certificate vault is using.
    See below for examples on how to acquire this.
  * `VAULT_TOKEN` A valid vault access token,
    see [auth token](https://www.vaultproject.io/docs/auth/token.html), default: `myroot`.
  * `VAULT_PKI_PATH` The path the PKI backend is mounted on in vault, default: `jwt_ca`.
  * `VAULT_PKI_ROLE` The vault PKI role used to generate new certificates, default: `trusted_service`.
  * `VAULT_PKI_CN`, `The common name, CN, used when generating new certificates, default: `trusted_service`.
  * `VAULT_EXPIRE_MARGIN` The time, in seconds, before certificate expiry to generate a new, 60.

#### Acquire b64 SHA256 fingerprint

If vault is setup according to below instructions, this below command will
print relevant data of the `https` certificate is uses.

```
docker run --rm \
  --volumes-from vault-tls \
  -e CERT_PATH=/var/tls \
  -e COMMON_NAME=vault \
  -e SUBJECT='/C=SE/O=Dev' \
  -e VALID_DAYS=365 \
  plugandtrade/certificate-generator:1.0.0
```

In the general case, the following command will produce the b64 sha256 fingerprint.
It requires openssl. Replace `localhost:8200` with host and port of your vault instance.

```
echo | \
  openssl s_client -showcerts -connect localhost:8200 2>/dev/null | \
  openssl x509 -outform der | \
  openssl dgst -sha256 -binary | \
  openssl base64 -A
```


#### Setup vault for development

Create a docker container with a new certificate pair. It will be mounted into the vault container.
```
docker run \
  --name vault-tls \
  -v /var/tls \
  -e CERT_PATH=/var/tls \
  -e COMMON_NAME=vault \
  -e SUBJECT='/C=SE/O=Dev' \
  -e VALID_DAYS=365 \
  plugandtrade/certificate-generator:1.0.0
```

Start vault in interactive mode. It mounts the certificates from the previous
container and sets the root access token to `myroot`. To start vault
non-interactively replace `run -it --rm` with `run -d` on the first line.

```
docker run -it --rm \
  --name=vault \
  --cap-add=IPC_LOCK \
  -e 'VAULT_DEV_ROOT_TOKEN_ID=myroot' \
  -p 8200:8201 \
  --volumes-from vault-tls \
  -e 'VAULT_LOCAL_CONFIG={"listener":[{"tcp":{"address":"0.0.0.0:8201","tls_cert_file":"/var/tls/vault.crt","tls_key_file": "/var/tls/vault.key"}}]}' \
  vault
```

Configure vault PKI secret backend to be used as a certificate provider, see `priv/vault_bootstrap.json` for more
details. It mounts a PKI backend at `jwt_ca` with two roles, `trusted_service` and `another_trusted_service`. Each with
certificates valid for 5 minutes.

```
docker run --rm -a stdout -a stderr \
  --name vault-bootstrap \
  --link vault:vault \
  --volumes-from vault-tls \
  -v "$PWD/priv/vault_bootstrap.json":/vault_bootstrap/bootstrap.json \
  plugandtrade/vault-bootstrap:0.2.2 \
  bootstrap \
  --host https://vault:8201 \
  --capath /var/tls/vault.crt \
  --config /vault_bootstrap/bootstrap.json \
  --token myroot
```

Documentation can be generated with [ExDoc](https://github.com/elixir-lang/ex_doc)
and published on [HexDocs](https://hexdocs.pm). Once published, the docs can
be found at [https://hexdocs.pm/jwk_provider](https://hexdocs.pm/jwk_provider).

