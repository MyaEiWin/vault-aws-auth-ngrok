# AWS EC2 Authentication to a Local Vault Dev Server Through ngrok

Authenticate an AWS EC2 instance to a local HashiCorp Vault dev server through ngrok and retrieve KV secrets using AWS IAM authentication.


## Summary

This lab demonstrates an end-to-end AWS IAM authentication flow between an Amazon Linux 2023 EC2 instance and a Vault dev server running on a local laptop.

The completed setup does the following:

1. Runs Vault dev mode locally with the KV v2 engine mounted at `secret/`.
2. Stores a demonstration value at `secret/demo`.
3. Exposes the local Vault API temporarily through an ngrok HTTPS tunnel.
4. Attaches the AWS IAM role `ec2-vault-role` to the EC2 instance as its instance profile.
5. Uses the Vault AWS auth method mounted at `aws-vault-policy-1/` to verify the EC2 instance's IAM identity.
6. Issues the EC2 instance a short-lived Vault token with `aws-secret-policy-1`.
7. Allows that token to read `secret/demo` without copying the Vault root token or permanent AWS access keys to EC2.

The successful command on EC2 is:

```bash
vault kv get -field=message secret/demo
```

Expected result:

```text
Hello from local Vault
```

The `app1/` AWS secrets engine is optional and separate from this KV retrieval flow. It is needed only when Vault must generate temporary AWS credentials for an application.

> [!WARNING]
> This setup is for short-lived learning and testing only. Vault dev mode is insecure, starts automatically unsealed, uses in-memory storage, and loses all data when stopped. Never put production credentials or other real secrets in this server.

## How the setup works

```text
Amazon Linux 2023 EC2 instance
  IAM instance profile: ec2-vault-role
                  |
                  | AWS IAM login over HTTPS
                  v
      https://<random-name>.ngrok-free.dev
                  |
                  | ngrok tunnel
                  v
        http://127.0.0.1:8200
                  |
                  v
          Local Vault dev server
                  |
                  | aws-secret-policy-1
                  v
          KV secret: secret/demo
```

You will use three terminals:

1. Terminal 1 runs the Vault server.
2. Terminal 2 runs Vault client commands.
3. Terminal 3 runs the ngrok tunnel.

## Prerequisites

- HashiCorp Vault installed
- An [ngrok account](https://dashboard.ngrok.com/signup)
- `sudo` access if ngrok needs to be installed

Check Vault:

```bash
vault version
```

Vault is already installed on this machine and was detected as version `1.20.0` when this guide was created.

## Step 1: Open the project directory

```bash
cd /vault-local-ngrok
```

## Step 2: Install ngrok

Check whether it is already installed:

```bash
ngrok version
```

If the command is not found, install ngrok with Snap:

```bash
sudo snap install ngrok
```

Verify the installation:

```bash
ngrok version
```

If Snap is unavailable, follow the [official ngrok Linux installation instructions](https://ngrok.com/download/linux).

## Step 3: Add the ngrok authtoken

1. Sign in to the [ngrok dashboard](https://dashboard.ngrok.com/).
2. Find and copy your ngrok authtoken. (Settings -> Authtokens)
3. Add it to the ngrok configuration:

```bash
ngrok config add-authtoken "YOUR_NGROK_AUTHTOKEN"

Authtoken saved to configuration file: /home/xx/snap/ngrok/429/.config/ngrok/ngrok.yml
```

Replace `YOUR_NGROK_AUTHTOKEN` with the value from your dashboard.

## Step 4: Start Vault in Terminal 1

Open the first terminal and run:

```bash
cd /home/mya/Documents/ACE/vault-local-ngrok
vault server -dev -dev-listen-address="127.0.0.1:8200"

```

Vault prints output containing an unseal key and a root token:

```text
2026-09-19T02:22:00.114-0400 [INFO]  core: post-unseal setup complete
2026-09-19T02:22:00.114-0400 [INFO]  core: vault is unsealed
2026-09-19T02:22:00.122-0400 [INFO]  core: successful mount: namespace="" path=secret/ type=kv version="v0.24.0+builtin"
WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variables:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: xxx/xICRuI=
Root Token: hvs.xxxxx

Development mode should NOT be used in production installations!
```

Temporarily save the root token somewhere private. Do not add it to this project or commit it to Git.

Leave Terminal 1 running. Stopping this process stops Vault and deletes all data stored in the dev server.

## Step 5: Test Vault locally in Terminal 2

Open a second terminal and configure the Vault CLI:

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="PASTE_THE_ROOT_TOKEN_FROM_TERMINAL_1"
```

Confirm that Vault is running:

```bash
vault status
vault token lookup
vault login
```

Expected status values include:

```text

vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.20.0
Build Date      2025-06-23T10:21:30Z
Storage Type    inmem
Cluster Name    vault-cluster-ed9511db
Cluster ID      10626150-4c76-2ec4-f2e9-0ff1c18a897f
HA Enabled      false


vault token lookup
Key                 Value
---                 -----
accessor            REDACTED_TOKEN_ACCESSOR
creation_time       1789799171
creation_ttl        0s
display_name        root
entity_id           n/a
expire_time         <nil>
explicit_max_ttl    0s
id                  hvs.REDACTED_ROOT_TOKEN
meta                <nil>
num_uses            0
orphan              true
path                auth/token/root
policies            [root]
ttl                 0s
type                service
```

Create a harmless demonstration secret:

```bash
vault kv put secret/demo message="Hello from local Vault"
```

```
== Secret Path ==
secret/data/demo

======= Metadata =======
Key                Value
---                -----
created_time       2026-09-19T06:28:00.225502849Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

Read it back:

```bash
vault kv get secret/demo
```

```text
== Secret Path ==
secret/data/demo

======= Metadata =======
Key                Value
---                -----
created_time       2026-09-19T06:28:00.225502849Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

===== Data =====
Key        Value
---        -----
message    Hello from local Vault
```

The local Vault UI is available at:

```text
http://127.0.0.1:8200/ui
```

Select the **Token** login method and enter the root token printed in Terminal 1.

## Step 6: Start ngrok in Terminal 3

Open a third terminal and run:

```bash
ngrok http http://127.0.0.1:8200
```

ngrok displays a forwarding URL similar to:

```text                                                                                                                  
Session Status                online                                                                                
Account                       aa-hello (Plan: Free)                                                                 
Version                       3.39.11                                                                               
Region                        United States (us)                                                                    
Latency                       38ms                                                                                  
Web Interface                 http://127.0.0.1:4040                                                                 
Forwarding                    https://xxxxx.ngrok-free.dev -> http://127.0.0.1:8200               
                                                                                                                    
Connections                   ttl     opn     rt1     rt5     p50     p90                                           
                              0       0       0.00    0.00    0.00    0.00                                          
                                                                                   
```

In this example, the public Vault address is:

```text
https://xxxxx.ngrok-free.dev
```

Keep Terminal 3 running. A free ngrok URL normally changes when the tunnel is restarted.

## Step 7: Test public access

Replace `https://xxxxx.ngrok-free.dev` in every command below with the hostname shown in Terminal 3.

Test the Vault health endpoint without a token:

```bash
curl https://YOUR-NGROK-HOSTNAME.ngrok-free.dev/v1/sys/health
```

The JSON response should show that Vault is initialized and not sealed.

Test authenticated access using the Vault CLI:

```bash
VAULT_ADDR="https://YOUR-NGROK-HOSTNAME.ngrok-free.dev" \
VAULT_TOKEN="PASTE_THE_ROOT_TOKEN" \
vault kv get secret/demo
```

The public Vault UI is available at:

```text
https://YOUR-NGROK-HOSTNAME.ngrok-free.dev/ui
```

## Step 8: Connect the EC2 instance to Vault

The EC2 instance uses the AWS IAM role `ec2-vault-role`. Attach that role to the instance as its IAM instance profile.

The Vault server continues to run on the laptop. Do **not** start another Vault server on EC2. The EC2 instance only needs a Vault client. The Vault CLI is the easiest client for this lab; Vault Agent, an application SDK, or the HTTP API are alternatives.

### 8.1 Verify the existing Vault mounts

On the laptop, use the dev root token:

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="PASTE_THE_ROOT_TOKEN_FROM_TERMINAL_1"

vault auth list
vault secrets list
```

This lab uses these existing paths:

```text
aws-vault-policy-1/    AWS authentication method
app1/                 AWS secrets engine
secret/               KV v2 secrets engine
```

Do not enable another AWS secrets engine at `aws-secrets/`; use the existing `app1/` mount.

### 8.2 Create the Vault policy for the EC2 workload

The EC2 workload only needs to read the demonstration KV secret:

```bash
vault policy write aws-secret-policy-1 - <<'EOF'
path "secret/data/demo" {
  capabilities = ["read"]
}

path "secret/metadata/demo" {
  capabilities = ["read"]
}
EOF
```

Verify it:

```bash
vault policy read aws-secret-policy-1
```

### 8.3 Configure the AWS authentication role

Set the AWS account ID and the ARN of the IAM role attached to the EC2 instance:

```bash
export AWS_ACCOUNT_ID="123456789012"
export EC2_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/ec2-vault-role"
```

Create a Vault AWS-auth role at the existing `aws-vault-policy-1/` mount:

```bash
vault write auth/aws-vault-policy-1/role/ec2-vault-role \
  auth_type="iam" \
  bound_iam_principal_arn="$EC2_ROLE_ARN" \
  resolve_aws_unique_ids=false \
  policies="aws-secret-policy-1" \
  token_ttl="30m" \
  token_max_ttl="1h"
```

Example output:

```text
Success! Data written to: auth/aws-vault-policy-1/role/ec2-vault-role
```

Read the configuration back:

```bash
vault read auth/aws-vault-policy-1/role/ec2-vault-role
```

Example output:

```text
Key                               Value
---                               -----
allow_instance_migration          false
auth_type                         iam
bound_account_id                  []
bound_ami_id                      []
bound_ec2_instance_id             <nil>
bound_iam_instance_profile_arn    []
bound_iam_principal_arn           [arn:aws:iam::xxxxxx:role/ec2-vault-role]
bound_iam_principal_id            []
bound_iam_role_arn                []
bound_region                      []
bound_subnet_id                   []
bound_vpc_id                      []
disallow_reauthentication         false
inferred_aws_region               n/a
inferred_entity_type              n/a
policies                          [aws-secret-policy-1]
resolve_aws_unique_ids            false
role_id                           REDACTED
role_tag                          n/a
token_bound_cidrs                 []
token_explicit_max_ttl            0s
token_max_ttl                     1h
token_no_default_policy           false
token_num_uses                    0
token_period                      0s
token_policies                    [aws-secret-policy-1]
token_ttl                         30m
token_type                        default
```

This dev lab disables AWS unique-ID resolution because the local AWS auth mount has no AWS client credentials. This binds the role by ARN instead of its immutable AWS principal ID. For production, configure a dedicated AWS identity with `iam:GetRole` permission and keep unique-ID resolution enabled. Do not attach a broad `sts:*` policy on `*`.

### 8.4 Install a Vault client on EC2

SSH to the EC2 instance and identify its operating system:

```bash
cat /etc/os-release
```

Check whether the Vault CLI is already installed:

```bash
vault version
```

If it is not installed, use the commands for the EC2 instance's operating system.

For Ubuntu or Debian:

```bash
sudo apt-get update
sudo apt-get install -y wget gpg

wget -O - https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update
sudo apt-get install -y vault
```

For this lab's **Amazon Linux 2023** EC2 instance, use `dnf` instead of `apt-get`:

```bash
sudo dnf install -y dnf-plugins-core shadow-utils

sudo dnf config-manager \
  --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo

sudo dnf install -y vault
```

Amazon Linux does not provide `apt-get`. If `sudo apt-get update` returns `command not found`, use the Amazon Linux 2023 commands above.

Verify the client:

```bash
vault version
```

These commands come from the [official Vault installation guide](https://developer.hashicorp.com/vault/install).

Installing the Vault package provides the CLI needed by this lab. The Vault server must remain on the laptop, so do not run `vault server -dev` or start the `vault` systemd service on the EC2 instance.

### 8.5 Authenticate from EC2 using its IAM role

First, verify through EC2 Instance Metadata Service v2 that an IAM role is attached:

```bash
IMDS_TOKEN=$(curl -fsS -X PUT \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300" \
  http://169.254.169.254/latest/api/token)

curl -fsS \
  -H "X-aws-ec2-metadata-token: $IMDS_TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

The response should be:

```text
ec2-vault-role
```

If AWS CLI is installed, verify the AWS identity as well:

```bash
aws sts get-caller-identity
```

The returned ARN should contain `assumed-role/ec2-vault-role/`.

Set the public ngrok address shown on the laptop in Terminal 3:

```bash
export VAULT_ADDR="https://YOUR-NGROK-HOSTNAME.ngrok-free.dev"
```

Check network access from EC2 to the laptop's Vault tunnel:

```bash
curl -fsS "$VAULT_ADDR/v1/sys/health"
```

The response should include `"initialized":true` and `"sealed":false`. If this fails, confirm that both Vault and ngrok are still running on the laptop and that `VAULT_ADDR` contains the current ngrok hostname.

Log in to Vault. The Vault CLI automatically uses the instance-profile credentials provided by EC2 Instance Metadata Service:

```bash
vault login \
  -method=aws \
  -path=aws-vault-policy-1 \
  region=auto \
  role=ec2-vault-role
```

Do not copy the laptop's Vault root token to EC2. A successful AWS login returns a short-lived Vault token with `aws-secret-policy-1` attached.

If an IAM server ID header was configured on the Vault AWS auth mount, include the exact configured value during login:

```bash
vault login \
  -method=aws \
  -path=aws-vault-policy-1 \
  region=auto \
  header_value="YOUR-CONFIGURED-SERVER-ID" \
  role=ec2-vault-role
```

Example successful login output:

```text
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                      Value
---                      -----
token                    hvs.REDACTED
token_accessor           REDACTED
token_duration           30m
token_renewable          true
token_policies           ["aws-secret-policy-1" "default"]
identity_policies        []
policies                 ["aws-secret-policy-1" "default"]
token_meta_auth_type     iam
token_meta_role_id       REDACTED
token_meta_account_id    REDACTED
```

Do not add `header_value` when the auth mount was not configured to require one.

Verify the Vault token and attached policy:

```bash
vault token lookup
```

Example output:

```text
Key                 Value
---                 -----
accessor            REDACTED
creation_time       1789805184
creation_ttl        30m
display_name        aws-vault-policy-1-ec2-vault-role/i-xxxxx
entity_id           REDACTED
expire_time         2026-09-19T04:36:24.846828216-04:00
explicit_max_ttl    0s
id                  hvs.REDACTED
issue_time          2026-09-19T04:06:24.846832739-04:00
meta                map[account_id:REDACTED auth_type:iam role_id:REDACTED]
num_uses            0
orphan              true
path                auth/aws-vault-policy-1/login
policies            [aws-secret-policy-1 default]
renewable           true
ttl                 28m37s
type                service
```

The output should list `aws-secret-policy-1` under `policies`.

### 8.6 Retrieve the KV secret from EC2

```bash
vault kv get secret/demo
```

To return only the `message` value:

```bash
vault kv get -field=message secret/demo
```

The policy intentionally prevents this EC2 identity from changing or deleting the secret.

After testing, remove the cached Vault token from the EC2 instance:

```bash
vault token revoke -self
unset VAULT_ADDR IMDS_TOKEN
```

## Step 9: Optional AWS secrets-engine integration

The `app1/` AWS secrets engine is not required for EC2 to read `secret/demo`. AWS authentication gives the instance a Vault token; the KV engine returns the application secret.

Use `app1/` only if the workload also needs Vault to generate a second set of temporary AWS credentials. This lab reuses `ec2-vault-role` and the existing `EC2_ROLE_ARN` variable for simplicity.

> [!CAUTION]
> Reusing one IAM role for both the EC2 instance profile and the AWS secrets engine mixes two responsibilities. A separate target role is recommended outside this temporary lab. When the same role is reused, its trust policy must allow both the EC2 service and the AWS identity configured for Vault's `app1/` engine.

After configuring `app1/config/root` with a dedicated Vault AWS identity, create the secrets-engine role:

```bash
export AWS_ACCOUNT_ID="123456789012"
export EC2_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/ec2-vault-role"

vault write app1/roles/ec2-operator \
  credential_type="assumed_role" \
  role_arns="$EC2_ROLE_ARN"
```

The AWS identity configured at `app1/config/root` must have `sts:AssumeRole` permission for `$EC2_ROLE_ARN`. The trust policy of `ec2-vault-role` must also trust that identity. Its existing EC2 service trust must remain in place so AWS can continue attaching the role to the instance.

Add credential-generation permission to `aws-secret-policy-1` only if the EC2 workload needs it:

```hcl
path "app1/sts/ec2-operator" {
  capabilities = ["update"]
}
```

Then generate temporary credentials from EC2:

```bash
vault write app1/sts/ec2-operator ttl="30m"
```

The AWS IAM policies attached to `ec2-vault-role` determine which EC2 API operations those generated credentials can perform.

## Step 10: Shut everything down

When testing is finished:

1. Press `Ctrl+C` in Terminal 3 to stop ngrok.
2. Press `Ctrl+C` in Terminal 1 to stop Vault.
3. Remove Vault variables from every client terminal:

```bash
unset VAULT_ADDR VAULT_TOKEN
```

Stopping the Vault dev server permanently deletes its secrets and tokens because its storage is in memory.

## Troubleshooting

### `connection refused` on port 8200

Make sure Terminal 1 is still running, then check the local health endpoint:

```bash
curl http://127.0.0.1:8200/v1/sys/health
```

### Vault tries to use HTTPS locally

The server in this guide uses local HTTP. Set the exact local address:

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
```

Do not use `https://127.0.0.1:8200` with this command-line configuration.

### `permission denied`

Check which token is active:

```bash
vault token lookup


```

The restricted token can access only `secret/demo`; denial for other paths is expected.

### ngrok says authentication failed

Add the correct authtoken from your ngrok dashboard:

```bash
ngrok config add-authtoken "YOUR_NGROK_AUTHTOKEN"
```

### The public URL stopped working

Both the Vault process and the ngrok process must remain running. If ngrok was restarted, copy its new public hostname and update `VAULT_ADDR`.

### Shell prints `iexport: command not found`

This machine previously displayed:

```text
/home/mya/.bashrc: line 1: iexport: command not found
```

Inspect the first line of `~/.bashrc`. If `iexport` is a typo, change it to the intended shell command, likely `export`.

## Security notes

- Never use Vault dev mode for production or persistent data.
- Never expose real secrets through this demonstration server.
- Never share the root token.
- Prefer a restricted token with a short TTL for remote testing.
- Stop ngrok immediately after testing.
- ngrok terminates the public TLS connection and is part of the trusted path.
- Request-inspection features can expose the `X-Vault-Token` header; handle ngrok logs and dashboards as sensitive data.
- For a persistent deployment, use HCP Vault or a properly hardened, TLS-enabled Vault server.

## Official documentation

- [HashiCorp: Vault dev server mode](https://developer.hashicorp.com/vault/docs/concepts/dev-server)
- [HashiCorp: Set up a Vault dev server](https://developer.hashicorp.com/vault/tutorials/get-started/setup)
- [ngrok: Linux installation](https://ngrok.com/download/linux)
