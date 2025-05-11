
# Migrate Existing Vault Cluster to AWS KMS Auto-Unseal
This process enables automatic unsealing using AWS KMS, replacing manual Shamir key-based unsealing.

## Recommended Pre-Migration Backup Steps
- Backup Existing Unseal Keys (Shamir)
- Backup Vault Configuration Files
    ```
    sudo cp -r /etc/vault.d /etc/vault.d.bak
    sudo cp -r /var/vault/raft /var/vault/raft.bak
    ```
- View Raft Cluster Peers (from any node)
  ```json
    vault operator raft list-peers

    Node       Address               State       Voter
    ----       -------               -----       -----
    vault-1    172.17.18.250:8201    leader      true
    vault-2    172.17.18.251:8201    follower    true
    vault-3    172.17.18.252:8201    follower    true
  ```
## Step 1: Create an AWS IAM User
- Go to IAM → Users → Create User (e.g., vault)
- No policies are required yet
- Save the Access Key and Secret Key

## Steps 2: Create a KMS Key
- Go to KMS → Create Key
- **Choose:**
- Key Type: Symmetric
- Key Usage: Encrypt and Decrypt
- In the key access permissions, add the Vault IAM user (e.g., vault)
- Finalize key creation

## Steps 3: Set KMS Key Policy
Edit the KMS key policy to allow the Vault IAM user to manage and use the key

```json
{
  "Id": "key-consolepolicy-3",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::605134426044:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow use of the key",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::605134426044:user/vault"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow attachment of persistent resources",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::605134426044:user/vault"
      },
      "Action": [
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": "true"
        }
      }
    }
  ]
}
```

## Steps 4: Attach IAM Policy to Vault User
Go to IAM → Users → vault → Add Inline Policy
```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "AllowVaultToUseKMS",
			"Effect": "Allow",
			"Action": [
				"kms:Encrypt",
				"kms:Decrypt",
				"kms:DescribeKey",
				"kms:GenerateDataKey"
			],
			"Resource": "arn:aws:kms:us-east-1:123456789012:key/abcd1234-5678-efgh-ijkl-9876mn0pqrs1"
		}
	]
}
```

## Step 5: Update Vault Configuration File of each node, First do Active/Master Node

Edit `/etc/vault.d/vault.hcl` on each Vault node to include the AWS KMS seal configuration:


`sudo vim /etc/vault.d/vault.hcl`

```json
seal "awskms" {
  region = "eu-north-1"
  kms_key_id = "arn:aws:kms:eu-north-1:605134426044:key/7fec9fd9-d04e-40ab-a92b-d73d3f3e2b9f"
  access_key = "vault user access key"
  secret_key = "vault user secret key"
}
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
## Step 6: Restart Vault with New Seal Configuration

`sudo systemctl stop vault.service`

`sudo systemctl start vault.service`

 ## Step 7: Migrate Seal (Only Once Per Node)

`export VAULT_ADDR=http://127.0.0.1:8200`

`vault operator unseal -migrate Z+v92NmIZRo2DrMhbXkyZfa/zsMdJ+6awR+/3qOHvfrj`

`vault operator unseal -migrate lsTY7Pq7w044OGkrai/55CNTkql0PfGaaaSSY3FbmHPt`

`vault operator unseal -migrate 28TMSYh5mx0u3tMJLt/qnjF/D42jgyeKaB7pUdjhfopt`

## Step 8: Verify Cluster State

`vault status`

`vault operator raft list-peers`


**Note** On each node:
- Update the config (vault.hcl)
- Restart the Vault service
- Run vault operator unseal -migrate ... with 3 Shamir keys
- Make sure `vault status`
  ```json
    Seal Type                awskms
    Recovery Seal Type       shamir
    Initialized              true
    Sealed                   false
  ```