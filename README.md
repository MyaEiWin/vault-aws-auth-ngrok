# Expose a Local Vault Dev Server with ngrok

This guide runs HashiCorp Vault in local development mode and makes it temporarily reachable from the internet through an ngrok HTTPS URL.

> [!WARNING]
> This setup is for short-lived learning and testing only. Vault dev mode is insecure, starts automatically unsealed, uses in-memory storage, and loses all data when stopped. Never put production credentials or other real secrets in this server.

## How the setup works

```text
Internet client
      |
      | HTTPS
      v
https://<random-name>.ngrok-free.app
      |
      | ngrok tunnel
      v
http://127.0.0.1:8200
      |
      v
Local Vault dev server
```

You will use three terminals:

1. Terminal 1 runs the Vault server.
2. Terminal 2 runs Vault client commands.
3. Terminal 3 runs the ngrok tunnel.

## Prerequisites

- A Linux laptop
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
cd /home/mya/Documents/ACE/19Sept2026
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

Authtoken saved to configuration file: /home/mya/snap/ngrok/429/.config/ngrok/ngrok.yml
```

Replace `YOUR_NGROK_AUTHTOKEN` with the value from your dashboard.

> [!CAUTION]
> Do not put the ngrok authtoken in this README, source control, screenshots, or chat messages.

## Step 4: Start Vault in Terminal 1

Open the first terminal and run:

```bash
cd /home/mya/Documents/ACE/19Sept2026
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
https://abc123.ngrok-free.app
```

Keep Terminal 3 running. A free ngrok URL normally changes when the tunnel is restarted.

## Step 7: Test public access

Replace `abc123.ngrok-free.app` in every command below with the hostname shown in Terminal 3.

Test the Vault health endpoint without a token:

```bash
curl https://abc123.ngrok-free.app/v1/sys/health
```

The JSON response should show that Vault is initialized and not sealed.

Test authenticated access using the Vault CLI:

```bash
VAULT_ADDR="https://abc123.ngrok-free.app" \
VAULT_TOKEN="PASTE_THE_ROOT_TOKEN" \
vault kv get secret/demo
```

The public Vault UI is available at:

```text
https://abc123.ngrok-free.app/ui
```

## Step 8: Create a restricted internet token

Do not give an internet client the root token. Create a short-lived token that can access only the demonstration secret.

Return to Terminal 2 and make sure it is using the local Vault address and root token:

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="PASTE_THE_ROOT_TOKEN_FROM_TERMINAL_1"
```

Create a policy named `internet-demo`:

```bash
vault policy write internet-demo - <<'EOF'
path "secret/data/demo" {
  capabilities = ["create", "update", "read"]
}

path "secret/metadata/demo" {
  capabilities = ["read"]
}
EOF
```

Create a token that expires after 30 minutes:

```bash
vault token create \
  -policy="internet-demo" \
  -ttl="30m" \
  -explicit-max-ttl="30m"
```

The output contains a new token beginning with `hvs.`. Give the remote test client this restricted token instead of the root token.

Test the restricted token through ngrok:

```bash
export VAULT_ADDR="https://abc123.ngrok-free.app"
export VAULT_TOKEN="PASTE_THE_RESTRICTED_TOKEN"

vault kv get secret/demo
```

Test the HTTP API directly:

```bash
curl \
  -H "X-Vault-Token: PASTE_THE_RESTRICTED_TOKEN" \
  https://abc123.ngrok-free.app/v1/secret/data/demo
```

Do not put a Vault token in a URL query string.

## Step 9: Connect a remote application

Configure the remote test application with these two values:

```bash
export VAULT_ADDR="https://abc123.ngrok-free.app"
export VAULT_TOKEN="PASTE_THE_RESTRICTED_TOKEN"
```

Use the current ngrok address and the restricted token created in Step 8. Do not use the local `127.0.0.1` address on the remote machine.

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
