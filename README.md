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
2. Copy the username (`12345678\|accountname`) and the long token  

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
│   └── aap_tokens.yml          # entry point
├── roles/
│   └── aap_token_mgt/
│       ├── defaults/main.yml
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── setup.yml       # first-time write
│       │   └── keepalive.yml   # daily refresh + re-assert
│       └── meta/main.yml
├── group_vars/
│   └── all/
│       └── vault.yml.example   # copy → vault.yml, then encrypt
├── requirements.yml            # ansible.hub, ansible.controller, infra.aap_configuration
└── README.md
```

---

## Quick start

```bash
# Install dependencies
ansible-galaxy collection install -r requirements.yml

# Secrets
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
# edit vault.yml with real values
ansible-vault encrypt group_vars/all/vault.yml

# First run – write everything into AAP
ansible-playbook playbooks/aap_tokens.yml --tags setup --ask-vault-pass

# Thereafter – keep tokens alive (cron / AAP schedule)
ansible-playbook playbooks/aap_tokens.yml --tags keepalive --ask-vault-pass
```

### Suggested schedule

```cron
# Daily at 03:15
15 3 * * *  ansible-playbook /opt/infra-aap-token-mgt/playbooks/aap_tokens.yml \
              --tags keepalive \
              --vault-password-file /etc/ansible/vault_pass
```

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
