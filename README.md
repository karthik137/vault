### Vault overview

Manage secrets and protect sensitive data.

Secure store and tightly control access to tokens, passwords, certificats, encryption keys using UI, CLI or HTTP API.


#### Vault walkthrough

##### Install vault

```
$ wget https://releases.hashicorp.com/vault/1.1.2/vault_1.1.2_linux_amd64.zip

$ unzip vault_1.1.2_linux_amd64.zip

$ sudo mv ./vault /usr/bin
```


##### Check vault installation

```
 vault --help
Usage: vault <command> [args]

Common commands:
    read        Read data and retrieves secrets
    write       Write data, configuration, and secrets
    delete      Delete secrets and configuration
    list        List data or secrets
    login       Authenticate locally
    agent       Start a Vault agent
    server      Start a Vault server
    status      Print seal and HA status
    unwrap      Unwrap a wrapped secret

Other commands:
    audit          Interact with audit devices
    auth           Interact with auth methods
    kv             Interact with Vault's Key-Value storage
    lease          Interact with leases
    namespace      Interact with namespaces
    operator       Perform operator-specific tasks
    path-help      Retrieve API help for paths
    plugin         Interact with Vault plugins and catalog
    policy         Interact with policies
    print          Prints runtime configurations
    secrets        Interact with secrets engines
    ssh            Initiate an SSH session
    token          Interact with tokens

```

##### Starting the Server

Vault operates as a client/server application. The Vault server is the only piece of the Vault architecture that interacts with the data storage and backends. All operations done via the Vault CLI interact with the server over a TLS connection.

```
$ vault server -dev
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
                 Storage: inmem
                 Version: Vault v1.1.2
             Version Sha: 0082501623c0b704b87b1fbc84c2d725994bac54

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: 8eMpmqHppgB3MuJmfBzSL3xkhISBhNEVERLMKnVYzzo=
Root Token: s.diWCsyeKmo3fnbp7OuUV30Np

Development mode should NOT be used in production installations!


```


Now we need to do the following things


1) export VAULT_ADDR='http://localhost:8200'
2) export VAULT_DEV_ROOT_TOKEN_ID="<ROOT TOKEN>"


##### Verify server is running

```
$ vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.1.2
Cluster Name    vault-cluster-c4f9bdd9
Cluster ID      098f5874-52cd-6fb1-14bd-d3f08111543d
HA Enabled      false

```


##### Your first secret

One of the core features of Vault is the ability to read and write arbitrary secrets securely. 

Currently we are using vault dev server. So all the secrets are in the memory. 

Vault secrets can be queried using cli or http api.

Secrets written to vault are encrypted and then written back to the storage. So your storage never knows the real secret. Vault encrypts the value before storing it to the backend.

NOTE: Backennd does not have the ability to decrypt the vault secret.


###### Writing a secret 

his is done very simply with the 'vault kv'

```
$ vault kv put secret/hello name=kirito
Key              Value
---              -----
created_time     2019-05-14T16:13:41.598666343Z
deletion_time    n/a
destroyed        false
version          1

```


You can even write multiple pieces of data, if you want:

```
$ vault kv put secret/hello name=kirito lastname=kazuto
Key              Value
---              -----
created_time     2019-05-14T16:21:35.09505048Z
deletion_time    n/a
destroyed        false
version          2

```


###### Getting secrets from the vault

$ vault kv get secret/hello


```
$  vault kv get secret/hello 
====== Metadata ======
Key              Value
---              -----
created_time     2019-05-14T16:21:35.09505048Z
deletion_time    n/a
destroyed        false
version          2

====== Data ======
Key         Value
---         -----
lastname    kazuto
name        kirito

```


Output can be displayed inn json also

```
$ vault kv get -format=json secret/hello 
{
  "request_id": "e2c67001-9596-d67b-458f-0caa0ac81abd",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "data": {
      "lastname": "kazuto",
      "name": "kirito"
    },
    "metadata": {
      "created_time": "2019-05-14T16:21:35.09505048Z",
      "deletion_time": "",
      "destroyed": false,
      "version": 2
    }
  },
  "warnings": null
}

```

#### Secret Engines

```
$ vault secrets enable -path=kv kv

```

Vault by default provides kv secret engine and stores data at  /secret path. We can create new paths which can be used to store credentials/keys.

Vault provides various secret engines like aws secret engine annd database credentials.


#### AWS Secrets engine

Unlike the kv secrets where you had to put data into the store yourself, dynamic secrets are generated when they are accessed. Dynamic secrets do not exist until they are read, so there is no risk of someone stealing them or another client using the same secrets. Because Vault has built-in revocation mechanisms, dynamic secrets can be revoked immediately after use, minimizing the amount of time the secret existed.


AWS secrets engine needs to be enabled before they are used

```
$ vault secrets enable -path=aws aws
Success! Enabled the aws secrets engine at: aws/

```

Do the following steps:

1) Configure the aws secrets engine (Access_key, Secret_key and region)

2) Create a Role 

    Vault does know how to create a user. But it does not know the options/policy to be set. So vault provides a way to create role which takes the following parameters:

        {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1426528957000",
      "Effect": "Allow",
      "Action": ["ec2:*"],
      "Resource": ["*"]
    }
  ]
}

e.g.

```
$ vault write aws/roles/my-role \
        credential_type=iam_user \
        policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1426528957000",
      "Effect": "Allow",
      "Action": [
        "ec2:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF

```

######  Generating the Secret

$ vault read aws/creds/my-role

Key                Value
---                -----
lease_id           aws/creds/my-role/0bce0782-32aa-25ec-f61d-c026ff22106e
lease_duration     768h
lease_renewable    true
access_key         AKIAJELUDIANQGRXCTZQ
secret_key         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
security_token     <nil>


###### Revoking the secret

Vault automatically revokes the secret after 768 hours. 

```
vault lease revoke aws/creds/my-role/0bce0782-32aa-25ec-f61d-c026ff22106

```


#### Vault authentication

When starting the Vault server in dev mode, it automatically logs you in as the root user with admin permissions. In a non-dev setup, you would have had to authenticate first.


token auth method and the GitHub auth method are available.


Token authentication is enabled by default in Vault and cannot be disabled. When you start a dev server with vault server -dev, it prints your root token. The root token is the initial access token to configure Vault. It has root privileges, so it can perform any operation within Vault.


By default, this will create a child token of your current token that inherits all the same policies. The "child" concept here is important: tokens always have a parent, and when that parent token is revoked, children can also be revoked all in one operation. This makes it easy when removing access for a user, to remove access for all sub-tokens that user created as well.

####  Auth Methods (Authentication)

Authenication can be done many ways. But we need to enable it first


Vault provides token as default method. We can use github authenntication also.


e.g  $ vault login -method=github token=abcd1234

#### Policies (Autherization)

Policies in Vault control what a user can access. 

There are some built-in policies that cannot be removed. For example, the root and default policies are required policies and cannot be deleted. The default policy provides a common set of permissions and is included on all tokens by default. The root policy gives a token super admin permissions, similar to a root user on a linux machine.


e.g

```
path "secret/*" {
  capabilities = ["create"]
}
path "secret/foo" {
  capabilities = ["read"]
}

# Dev servers have version 2 of KV mounted by default, so will need these
# paths:
path "secret/data/*" {
  capabilities = ["create"]
}
path "secret/data/foo" {
  capabilities = ["read"]
}

```



##### Setting up production server/Main server


1) Install consul (Backend)

```

wget https://releases.hashicorp.com/consul/1.5.0/consul_1.5.0_linux_amd64.zip

```
```
unzip consul_1.5.0_linux_amd64.zip
Archive:  consul_1.5.0_linux_amd64.zip
  inflating: consul                  
```

```
sudo mv ./consul /usr/bin/
```



2) Create hcl configuration file


e.g 

```
storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
}

listener "tcp" {
 address     = "127.0.0.1:8200"
 tls_disable = 1
}


```


###### Start consul in dev mode

```
consul agent -dev
```

###### Start vault server

```
$ sudo vault server -config=config.hcl
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: true
                 Storage: consul (HA available)
                 Version: Vault v1.1.2
             Version Sha: 0082501623c0b704b87b1fbc84c2d725994bac54


```
###### Initializing the Vault 

After starting the vault we need to initialize it.


It is done by following command.

```
vault operator init
```

Initialization is the process configuring the Vault. This only happens once when the server is started against a new backend that has never been used with Vault before. 

e.g

```
$  vault operator init
Unseal Key 1: QaEr6E7LxKh2xJMzMIlsEcfVEak85/HXqCcm6TQ0ze7j
Unseal Key 2: RHFZoBPrIyQNhxcAjvo4EdtftEZgdNBBLB/kksbgvbsw
Unseal Key 3: jVzQaJORfmHgN3f2dPabFXH9vTW+qIuWWeYZ3oDWBbQp
Unseal Key 4: VPQ9e5x2g8I/KLO8s/qXpUZskKCtxjt995BYKxaBxvm/
Unseal Key 5: 4l4+y5NKzmvAhrfl0YKHSRtzUM4MuzUGuYhPSnaYEks9

Initial Root Token: s.WrfZ2IemMwEJO2akRibBx1n0

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of

```



###### Seal/Unseal 

Every initialized Vault server starts in the sealed state. From the configuration, Vault can access the physical storage, but it can't read any of it because it doesn't know how to decrypt it. The process of teaching Vault how to decrypt the data is known as unsealing the Vault.

Unsealing has to happen every time Vault starts. It can be done via the API and via the command line.

It uses Shamir's Secret Sharing algorithm.


```
$ vault operator unseal
Unseal Key (will be hidden): 
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       a331f463-648a-6ef1-5624-6534b6454d87
Version            1.1.2
HA Enabled         true


```

This needs to be done 3 times to reach the threshold

Finally, authenticate as the initial root token (it was included in the output with the unseal keys):


