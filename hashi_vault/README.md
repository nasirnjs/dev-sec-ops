

# What is HashiCorp Vault 🔐?

HashiCorp Vault is a powerful tool designed to manage secrets, such as API keys, passwords, certificates, and encryption keys. It provides:

- Secure storage of secrets with fine-grained access control.
- Dynamic secrets generation (e.g., database credentials).
- Data encryption as a service.
- Identity-based access via policies and authentication backends.
- Vault is commonly used in DevOps and cloud-native environments to centralize and secure secret management across distributed systems.

## When Vault starts, it is in a sealed state

- Vault can access its physical storage backend (such as file system, Consul, or cloud storage),
but cannot decrypt any data.
- No operations are allowed except checking Vault's status or attempting to unseal it.

## How Encryption Works in Vault
- Vault encrypts data using a data encryption key, which resides in memory only when the Vault is unsealed.

- This data key is encrypted and stored in the keyring, which is persisted in the storage backend.

- The keyring is protected by a master key.

- The master key is never stored directly. Instead, it is sharded into multiple key shares using Shamir's Secret Sharing algorithm.

- These key shares are known as unseal keys.

## Install Vault
There are several methods to install Vault, but the recommended way is to use HashiCorp's official APT repository. This ensures you get the latest stable and secure version with proper support. [Here](https://developer.hashicorp.com/vault/install) the step-by-step installation process for Vault on Ubuntu.

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault
```

`sudo systemctl start vault.service`

`sudo systemctl enable vault.service`

## Vault Server in Production

Vault configuration parameters [Referencfes](https://developer.hashicorp.com/vault/docs/configuration)

`sudo systemctl status vault.service`

`cat /usr/lib/systemd/system/vault.service`\
You will see configuration file location `ExecStart=/usr/bin/vault server -config=/etc/vault.d/vault.hcl`

`sudo vim /etc/vault.d/vault.hcl`

`vault version`


`vault status` 


`sudo journalctl -u vault` `Shift + G`

 
### How does vault Protect my Data

- Master Key
- Encryption Key
- Seal and Unseal
- Auto Unseal

`export VAULT_ADDR=http://127.0.0.1:8200`

## Create Configuration File

`sudo mkdir -p /var/vault/raft/data `

`sudo chown -R vault:vault /var/vault/raft`

```bash
ui = true
disable_mlock = true

storage "raft" {
  path    = "/var/vault/raft/data"
  node_id = "raft_node_id"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

# Set the cluster address for Raft storage
cluster_addr = "https://127.0.0.1:8201"

# Set the API address for client requests
api_addr = "http://127.0.0.1:8200"

#telemetry {
#  statsite_address = "127.0.0.1:8125"
#  disable_hostname = true
#}
```

Identify potential misconfigurations or issues that might prevent Vault from starting correctly.\
`vault operator diagnose -config /etc/vault.d/vault.hcl`


## Initialize Vault

`vault operator init`

## Unseal Vault
- To unseal Vault, a quorum of unseal keys (e.g., 3 out of 5) must be provided.
- Once enough key shares are entered, Vault reconstructs the master key, decrypts the keyring, and loads the data encryption key into memory.
- At this point, Vault becomes unsealed and fully operational.
  
`vault operator unseal IE29DVQVJwLfP4U1HpuktGv43fuLftaDIHn+eV2A4Kdn`

Login with root token 

`vault login <your-root-token>`


## Vault secrets engine
A secrets engine in Vault is a component that manages and handles specific types of secrets. It provides different methods of secret storage, management, and generation. Each secrets engine is designed for a specific use case, such as storing key-value pairs, generating dynamic credentials for databases, or managing encryption keys.


### What Secrets Engines Are Available?

Vault offers a wide range of secrets engines that cater to different use cases. Some are enabled by default, while others may need to be enabled manually, such as:

- **Key-Value (kv)**: Store key-value pairs.
- **Database**: Generate database credentials.
- **AWS**: Generate AWS IAM credentials.
- **PKI**: Manage and issue TLS/SSL certificates.
- **Transit**: Perform encryption/decryption.
- **Identity**: Manage user and entity identities.
- **Vault Managed Storage**: Provides mechanisms for storing data securely.


### Now comes enabling secrets engines.

`vault secrets enable -path=prod-app kv`

`ault kv put prod-app/db username=admin password=secret123`

`ault kv get prod-app/db`

`vault kv get -format=json prod-app/db`

`vault kv put prod-app/api-keys stripe=sk_test_abc123 sendgrid=SG.abcd`

`vault kv get prod-app/api-keys`

`vault kv delete prod-app/api-keys`

`vault secrets list`


## Enable the AWS Secrets Engine
First, you need to enable the AWS secrets engine to generate dynamic AWS IAM credentials.

`vault secrets enable -path=aws aws`

`vault secrets list`

To list all AWS roles you've configured in HashiCorp Vault under the aws/ secrets engine.\
`vault list aws/roles`

`vault secrets disable aws`

## Configure the AWS Secrets Engine
Before configuring the engine, you need to create an IAM user for Vault. This user must have the necessary permissions (access key and secret key) required for Vault to perform operations such as creating IAM users, provisioning EC2, managing S3 buckets, and working with EKS resources.

`vault write aws/config/root access_key=<AWS_ACCESS_KEY> secret_key=<AWS_SECRET_KEY> region=<AWS_REGION>`

`vault write aws/config/root access_key=AKIAGHJT secret_key=4rIU2zK5DUZfhZvn region=us-east-2`

**Configure the Role**\
After configuring the AWS secrets engine, you'll want to configure roles that define the type of temporary AWS IAM credentials Vault should generate.

```bash
vault write aws/roles/vault-user-role \
    credential_type=iam_user \
    policy_arns=arn:aws:iam::aws:policy/AdministratorAccess \
    ttl=1h
```

**This means:**

- When someone requests credentials using this role (vault-user-role), Vault creates a real IAM user with the AdministratorAccess policy.
- That IAM user can then do anything in the AWS account (within its TTL).
- After the TTL expires, Vault (if configured properly) revokes and deletes the IAM user.


```
vault write aws/roles/my-ec2-role \
        credential_type=iam_user \
        policy_document=-EOF
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

To list all AWS roles you've configured in HashiCorp Vault under the aws/ secrets engine.\
`vault list aws/roles`

View the details of the admin-user-role using.\
`vault read aws/roles/admin-user-role`


## Vault policy
A Vault policy in HashiCorp Vault is a set of rules written in HCL (HashiCorp Configuration Language) or JSON that defines what actions (like read, write, delete, list, etc.) are allowed or denied for a given path in Vault.

**Purpose**
Vault policies are used to control access management—who can do what in Vault. These are applied to:
- Tokens
- Users
- Entities
- Roles (for auth methods like Kubernetes, AppRole, etc.)

`vault policy list`



## Build a HashiCorp Vault Cluster Manually

**Vault master**

`sudo vim /etc/vault.d/vault.hcl`

```bash
listener "tcp" {
  address          = "0.0.0.0:8200"
  cluster_address  = "172.17.18.250:8201"
  tls_disable      = 1  # Set to 0 and provide certs for production
}

api_addr     = "http://172.17.18.250:8200"
cluster_addr = "http://172.17.18.250:8201"

storage "raft" {
  path    = "/var/vault/raft/data"
  node_id = "vault-1"
}

ui = true
disable_mlock = true
```

`sudo systemctl start vault.service`

`sudo systemctl status vault`

`export VAULT_ADDR='http://127.0.0.1:8200'`

`vault operator init`

`vault operator unseal lP70cV8GS03q5nX+`

`vault login`

`vault operator raft list-peers`


**Vault-2**

`sudo vim /etc/vault.d/vault.hcl`

```bash
listener "tcp" {
  address          = "0.0.0.0:8200"
  cluster_address  = "172.17.18.251:8201"
  tls_disable      = 1
}

api_addr     = "http://172.17.18.251:8200"
cluster_addr = "http://172.17.18.251:8201"

storage "raft" {
  path    = "/var/vault/raft/data"
  node_id = "vault-2"
  retry_join {
    leader_api_addr = "http://172.17.18.250:8200"
  }
}

ui = true
disable_mlock = true
```

`sudo systemctl start vault.service`

`sudo systemctl status vault`

`export VAULT_ADDR='http://127.0.0.1:8200'`

`vault operator raft join http://172.17.18.250:8200`

`vault operator unseal lP70cV8GS03q5nX+=5nsPbpqTXQ8P`

`vault operator unseal Dc4jMi5l5zzOsYZ=sn+QadUzrB+SUo`

`vault operator unseal 0YwSNcUopB6sJiC=uNOWZwE07aTNs`


**Vault-3**

`sudo vim /etc/vault.d/vault.hcl`

```bash
listener "tcp" {
  address          = "0.0.0.0:8200"
  cluster_address  = "172.17.18.252:8201"
  tls_disable      = 1
}

api_addr     = "http://172.17.18.252:8200"
cluster_addr = "http://172.17.18.252:8201"

storage "raft" {
  path    = "/var/vault/raft/data"
  node_id = "vault-2"
  retry_join {
    leader_api_addr = "http://172.17.18.250:8200"
  }
}

ui = true
disable_mlock = true
```

`sudo systemctl start vault.service`

`sudo systemctl status vault`

`export VAULT_ADDR='http://127.0.0.1:8200'`

`vault operator raft join http://172.17.18.250:8200`

`vault operator unseal lP70cV8GS03q5nX+=5nsPbpqTXQ8P`

`vault operator unseal Dc4jMi5l5zzOsYZ=sn+QadUzrB+SUo`

`vault operator unseal 0YwSNcUopB6sJiC=uNOWZwE07aTNs`



- All nodes show Sealed: false
- Node 1: HA Mode: active
- Nodes 2 & 3: HA Mode: standby

```bash
# On Node 1
sudo systemctl stop vault

# On Node 2 or 3
vault status
# Should see one promote to leader within 15-30 seconds
```


