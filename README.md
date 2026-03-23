# Splunk Universal Forwarder Ansible Automation

This project is structured for API-driven execution with minimal runtime input. The caller only needs to provide Vault authentication, target hosts, and one remote access method. Package URLs, install paths, Splunk output targets, and Vault secret path conventions are defined in code.

## Runtime Inputs

Supported runtime inputs are loaded from environment variables in [defaults/main.yml](D:/GitHub%20Projects/Splunk%20Universal%20Forwarder/defaults/main.yml):

- `VAULT_TOKEN` or `VAULT_USERNAME` + `VAULT_PASSWORD`
- `TARGET_HOSTS`
- `SERVICE_ID` or `REMOTE_USER` + `REMOTE_PASSWORD`

## Project Layout

- `site.yml`: root entrypoint
- `playbooks/install_splunk_uf.yml`: staged workflow
- `defaults/main.yml`: runtime env-var mapping
- `group_vars/all.yml`: static platform configuration
- `roles/preflight`: validate and normalize inputs
- `roles/vault_auth`: authenticate to Vault
- `roles/resolve_connection`: resolve SSH key or password connection details
- `roles/prepare_hosts`: verify connectivity and derive OS-aware package info
- `roles/splunk_install`: install UF and enable the service
- `roles/splunk_config`: deploy predefined config templates
- `roles/splunk_validate`: verify service and forward-server status
- `roles/execution_summary`: emit a structured summary
- `templates/`: predefined Splunk config templates

## Example Usage

Password path:

```powershell
$env:VAULT_TOKEN = "s.xxxxx"
$env:TARGET_HOSTS = "server1.example.com,server2.example.com"
$env:REMOTE_USER = "svc_splunk"
$env:REMOTE_PASSWORD = "super-secret"
ansible-playbook -i inventory/hosts.yml site.yml
```

Service ID path:

```powershell
$env:VAULT_TOKEN = "s.xxxxx"
$env:TARGET_HOSTS = "server1.example.com"
$env:SERVICE_ID = "splunk-forwarder"
ansible-playbook -i inventory/hosts.yml site.yml
```

## Security Notes

- Vault tokens, passwords, and retrieved SSH keys are handled with `no_log: true`.
- SSH private key material from Vault is written to a temporary controller file only for the current run.
- Static platform settings stay in [group_vars/all.yml](D:/GitHub%20Projects/Splunk%20Universal%20Forwarder/group_vars/all.yml) and should be moved to protected environment-specific config as needed.
