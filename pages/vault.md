---
layout: default
---

## Setting up Hashicorp Vault to store and manage secrets

Vault is an extensive tool with both opensource and paid version to allow you to secure, store and tightly control access to tokens, passwords, certificates, encryption keys for protecting secrets and other sensitive data using a UI, CLI, or HTTP API.

#### Install vault

```bash
brew install hashicorp/tap/vault
```
#### verify installation

 ```bash
$ vault

Usage: vault <command> [args]

Common commands:
    read        Read data and retrieves secrets
    write       Write data, configuration, and secrets
    delete      Delete secrets and configuration
    list        List data or secrets
    login       Authenticate locally
    server      Start a Vault server
    status      Print seal and HA status
    unwrap      Unwrap a wrapped secret

Other commands:
    audit          Interact with audit devices
    auth           Interact with auth methods
    lease          Interact with leases
    operator       Perform operator-specific tasks
    path-help      Retrieve API help for paths
    policy         Interact with policies
    secrets        Interact with secrets engines
    ssh            Initiate an SSH session
    token          Interact with tokens
 ```

The dev server is a built-in, pre-configured server that is not very secure but useful for playing with Vault locally.

In development mode, Vault gets configured with several options that makes it convenient:

* Vault is already initialized with one key share (whereas in normal mode this has to be done explicitly and the number of key shares is 5 by default)
* the unseal key and the root token are displayed in the logs (please write down the root token, we will need it in the following step)
* Vault is unsealed
* in-memory storage
* TLS is disabled

#### start dev vault

To start the Vault dev server, run:

```bash
$ vault server -dev

==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: false, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.4.1

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: <xxxx>
Root Token: <xxxx>

Development mode should NOT be used in production installations!

==> Vault server started! Log data will stream in below:

```

#### export vault address and token to start using it

```bash
$ export VAULT_ADDR='http://127.0.0.1:8200'

$ export VAULT_TOKEN="xxxxxx"

```

#### check vault status
```bash
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.5.0
Cluster Name    vault-cluster-xxx
Cluster ID      xxxx
HA Enabled      false
```

#### Add a secret
```bash
vault kv put secret/app/hello/config secret-key=xxxx
```

We have defined a path secret/app/hello in Vault that we need to give access to from the Quarkus application.

#### create policy

Create a policy that gives read and write access to secret/app/hello and subpaths:

```bash
$ vault policy write my-policy my-policy.hcl
```


```bash
$ vault policy write my-policy -<<EOF
path "secret/data/app/hello/*" {
  capabilities = ["create", "update"]
}

EOF
```


#### vault auth enable and login using github authentication

```bash
$ vault auth enable github
```

set the organization

```bash
$ vault write auth/github/config organization=my-org
```

Configure the GitHub team wise authentication policy.

```bash
$ vault write auth/github/map/teams/engineering value=default,applications
```

Attempt to login with the github auth method.

```bash
$ vault login -method=github

GitHub Personal Access Token (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  xxx
token_accessor         xxx
token_duration         768h
token_renewable        true
token_policies         [default applications]
token_meta_org         my-org
token_meta_username    my-user
```


#### get the secret

```bash
vault kv get secret/app/hello/config
```

[back](../)