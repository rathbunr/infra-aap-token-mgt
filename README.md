# AAP Non-User Credential Management

Two-play Ansible playbook that manages the non-user Red Hat tokens required by Ansible Automation Platform (AAP).

## Plays

| Tag | Purpose |
|-----|---------|
| `setup` / `once` | One-time bootstrap – writes all tokens into AAP |
| `keepalive` / `daily` | Daily keep-alive – refreshes the offline token and re-asserts values |

## Where to obtain the tokens

1. **Offline token** (collection remotes – rh-certified etc.)  
   https://console.redhat.com/ansible/automation-hub/token  
   Click **Load Token** → copy the Offline token.  
   The keep-alive play refreshes it so it never expires (must be used at least once every 30 days).

2. **Registry Service Account** (registry.redhat.io container pulls)  
   https://access.redhat.com/terms-based-registry  
   Create **New Service Account** → copy the username (`12345678|myaccount`) and the long token.  
   No public API exists to rotate this; regenerate in the UI when needed.

3. **Hybrid Cloud Console Service Account** (Automation Analytics)  
   https://console.redhat.com/iam/service-accounts  
   Settings gear → **Service Accounts** → **Create service account**.  
   Copy **Client ID** + **Client Secret** (shown only once).  
   Add the service account to a User Access group that has the Automation Analytics role.

## Prerequisites

```bash
ansible-galaxy collection install ansible.hub ansible.controller
```

## Vault variables

Create an encrypted vault file (example provided as `vault.yml.example`):

```yaml
vault_aap_admin_password: "..."
vault_rh_offline_token: "eyJ..."
vault_rh_registry_token: "eyJ..."
vault_rh_analytics_client_id: "..."
vault_rh_analytics_client_secret: "..."
```

## Usage

```bash
# One-time bootstrap
ansible-playbook aap_tokens.yml --tags setup --ask-vault-pass

# Daily keep-alive (cron / AAP schedule)
ansible-playbook aap_tokens.yml --tags keepalive --ask-vault-pass
```

## Example cron

```cron
# Run every day at 03:15
15 3 * * * /usr/bin/ansible-playbook /opt/playbooks/aap_tokens.yml --tags keepalive --vault-password-file /etc/ansible/vault_pass
```

## References

- Offline token: https://console.redhat.com/ansible/automation-hub/token  
  https://access.redhat.com/articles/3626371
- Registry Service Account: https://access.redhat.com/terms-based-registry  
  https://access.redhat.com/articles/4259601
- Analytics Service Account: https://console.redhat.com/iam/service-accounts  
  https://access.redhat.com/articles/7112649
