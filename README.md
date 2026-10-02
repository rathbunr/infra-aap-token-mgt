# infra-aap-token-mgt

Manage **non-user** Red Hat tokens required by Ansible Automation Platform.

## What this does

| Token / credential | Official support used | Keep-alive |
|--------------------|----------------------|------------|
| Offline token → `rh-certified` collection remote | `infra.aap_configuration.hub_collection_remote` | Custom refresh (no official role) |
| Registry Service Account → `registry.redhat.io` | `infra.aap_configuration.hub_ee_registry` | Re-assert only |
| Analytics Client ID/Secret | `infra.aap_configuration.controller_settings` | Re-assert only |

## Where to obtain the tokens

1. **Offline token**  
   https://console.redhat.com/ansible/automation-hub/token  
   Click **Load Token**.

2. **Registry Service Account**  
   https://access.redhat.com/terms-based-registry  
   Create account → copy `numbers\|name` + token.

3. **Analytics Service Account**  
   https://console.redhat.com/iam/service-accounts  
   Create → copy Client ID + Secret (shown once).  
   Add the SA to a group with the Automation Analytics role.

## Layout

```
playbooks/aap_tokens.yml
roles/aap_token_mgt/
  defaults/main.yml
  tasks/{main,setup,keepalive}.yml
  meta/main.yml
group_vars/all/vault.yml.example
requirements.yml
```

## Prerequisites

```bash
ansible-galaxy collection install -r requirements.yml
```

## Usage

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
# edit vault.yml, then:
ansible-vault encrypt group_vars/all/vault.yml

# One-time bootstrap
ansible-playbook playbooks/aap_tokens.yml --tags setup --ask-vault-pass

# Daily keep-alive
ansible-playbook playbooks/aap_tokens.yml --tags keepalive --ask-vault-pass
```

## References

- Offline token: https://console.redhat.com/ansible/automation-hub/token · https://access.redhat.com/articles/3626371
- Registry SA: https://access.redhat.com/terms-based-registry · https://access.redhat.com/articles/4259601
- Analytics SA: https://console.redhat.com/iam/service-accounts · https://access.redhat.com/articles/7112649
- Official roles: [infra.aap_configuration](https://galaxy.ansible.com/ui/repo/published/infra/aap_configuration/)
