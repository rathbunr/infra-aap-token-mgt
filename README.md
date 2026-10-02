# infra-aap-token-mgt

Keep Ansible Automation Platform’s **non-user** Red Hat credentials healthy.

One playbook, two modes:

| Mode | Tag | What it does |
|------|-----|--------------|
| **Setup** | `setup` / `once` | Write all tokens into AAP the first time |
| **Keep-alive** | `keepalive` / `daily` | Refresh the offline token and re-assert everything |

---

## Credentials covered

| Credential | Used for | How it is applied | Official role |
|------------|----------|-------------------|---------------|
| **Offline token** | Collection remotes (`rh-certified`) | Written + refreshed every run | `hub_collection_remote` |
| **Registry Service Account** | Pulling images from `registry.redhat.io` | Written / re-asserted | `hub_ee_registry` |
| **Analytics Service Account** | Automation Analytics / Insights | Client ID + Secret in Controller settings | `controller_settings` |

> The offline-token **refresh** itself has no official role — that step is a simple `uri` call against Red Hat SSO.

---

## Where to get the tokens

### 1. Offline token (collections)

**URL:** [console.redhat.com/ansible/automation-hub/token](https://console.redhat.com/ansible/automation-hub/token)

1. Open the page  
2. Click **Load Token**  
3. Copy the offline token  

As long as this playbook runs at least once every 30 days, the token never expires.

### 2. Registry Service Account (containers)

**URL:** [access.redhat.com/terms-based-registry](https://access.redhat.com/terms-based-registry)

1. Click **New Service Account**  
2. Copy the username (`12345678|accountname`) and the long token  

There is no API to rotate this — regenerate in the UI when needed.

### 3. Analytics Service Account

**URL:** [console.redhat.com/iam/service-accounts](https://console.redhat.com/iam/service-accounts)

1. **Settings** → **Service Accounts** → **Create**  
2. Copy **Client ID** and **Client Secret** (shown only once)  
3. Add the service account to a User Access group that has the Automation Analytics role

---

## Repository layout

```text
.
├── playbooks/
│   ├── aap_tokens.yml                 # entry point (setup / keepalive)
│   └── configure_credential_type.yml  # one-time: create custom credential type
├── roles/
│   └── aap_token_mgt/
│       ├── defaults/main.yml
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── setup.yml
│       │   └── keepalive.yml
│       └── meta/main.yml
├── configs/
│   └── credential_types.yml           # CaC definition for the custom type
├── group_vars/
│   └── all/
│       └── vault.yml.example          # CLI-only secrets template
├── requirements.yml
└── README.md
```

---

## Quick start

### 1. Install collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2a. Run from AAP (recommended)

Create the custom credential type once:

```bash
ansible-playbook playbooks/configure_credential_type.yml \
  -e aap_hostname=https://aap.example.com \
  -e aap_username=admin \
  -e aap_password=secret \
  -e aap_validate_certs=false
```

Then in AAP:

1. **Credentials** → add a credential of type **AAP Token Management** and fill in the tokens  
2. Create a Project pointing at this repo  
3. Create a Job Template:
   - Playbook: `playbooks/aap_tokens.yml`
   - Credential: the one you just created
   - Job tags: `setup` (first run) or `keepalive` (daily)
4. Schedule the Job Template with tag `keepalive`

Secrets stay in AAP’s encrypted credential store — no vault file on disk.

### 2b. Run from CLI

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
# edit vault.yml with real values
ansible-vault encrypt group_vars/all/vault.yml

# First run
ansible-playbook playbooks/aap_tokens.yml --tags setup --ask-vault-pass

# Daily keep-alive
ansible-playbook playbooks/aap_tokens.yml --tags keepalive --ask-vault-pass
```

```cron
# Daily at 03:15
15 3 * * *  ansible-playbook /opt/infra-aap-token-mgt/playbooks/aap_tokens.yml \
              --tags keepalive \
              --vault-password-file /etc/ansible/vault_pass
```

---

## Custom credential type

Defined in `configs/credential_types.yml` and applied by `playbooks/configure_credential_type.yml`.

| Field | Secret | Purpose |
|-------|--------|--------|
| `aap_hostname` | no | Platform / Controller URL |
| `aap_username` | no | Admin user |
| `aap_password` | yes | Admin password |
| `aap_validate_certs` | no | TLS verification |
| `rh_offline_token` | yes | Collection remote offline token |
| `rh_registry_username` | no | Registry SA username (`id|name`) |
| `rh_registry_token` | yes | Registry SA token |
| `rh_analytics_client_id` | no | Analytics service account ID |
| `rh_analytics_client_secret` | yes | Analytics service account secret |

All fields are injected as `extra_vars` into the job.

---

## References

| Topic | Link |
|-------|------|
| Offline token page | https://console.redhat.com/ansible/automation-hub/token |
| Offline token docs | https://access.redhat.com/articles/3626371 |
| Registry Service Accounts | https://access.redhat.com/terms-based-registry |
| Registry auth article | https://access.redhat.com/articles/4259601 |
| Analytics Service Accounts | https://console.redhat.com/iam/service-accounts |
| Analytics config (AAP) | https://access.redhat.com/articles/7112649 |
| Official CaC roles | [infra.aap_configuration](https://galaxy.ansible.com/ui/repo/published/infra/aap_configuration/) |
