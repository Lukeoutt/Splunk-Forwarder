# Splunk Universal Forwarder Ansible Automation

This project is structured for API-driven execution with minimal runtime input. The caller only needs to provide Vault authentication, target hosts, and one remote access method. Package URLs, install paths, Splunk output targets, Vault secret path conventions, and temporary root SSH key generation are handled in code.

## Runtime Inputs

Supported runtime inputs are loaded from environment variables in [defaults/main.yml](D:/GitHub%20Projects/Splunk%20Universal%20Forwarder/defaults/main.yml):

- `VAULT_TOKEN` or `VAULT_USERNAME` + `VAULT_PASSWORD`
- `TARGET_HOSTS`
- `SERVICE_ID` or `REMOTE_USER` + `REMOTE_PASSWORD`

## Project Layout

- `site.yml`: root entrypoint
- `bootstrap_root_ssh.yml`: one-time root SSH bootstrap entrypoint
- `cleanup_root_ssh.yml`: one-time root SSH cleanup entrypoint
- `playbooks/install_splunk_uf.yml`: staged workflow
- `playbooks/bootstrap_root_ssh.yml`: staged root bootstrap workflow
- `playbooks/cleanup_root_ssh.yml`: staged root cleanup workflow
- `defaults/main.yml`: runtime env-var mapping
- `group_vars/all.yml`: static platform configuration
- `roles/preflight`: validate and normalize inputs
- `roles/vault_auth`: authenticate to Vault
- `roles/resolve_connection`: resolve SSH key or password connection details
- `roles/generate_ephemeral_root_key`: generate a temporary root SSH keypair on the runner when required
- `roles/prepare_hosts`: verify connectivity and derive OS-aware package info
- `roles/bootstrap_root_ssh`: install a root authorized key using direct root access or `sudo su -`
- `roles/promote_root_connection`: switch the execution group to direct root SSH
- `roles/cleanup_root_ssh`: remove the temporary root authorized key after the run
- `roles/cleanup_local_ephemeral_key`: delete the generated temporary keypair from the runner
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

Root SSH bootstrap:

```powershell
$env:VAULT_TOKEN = "s.xxxxx"
$env:TARGET_HOSTS = "server1.example.com"
$env:REMOTE_USER = "svc_splunk"
$env:REMOTE_PASSWORD = "super-secret"
ansible-playbook -i inventory/hosts.yml bootstrap_root_ssh.yml
```

Root SSH cleanup:

```powershell
$env:VAULT_TOKEN = "s.xxxxx"
$env:TARGET_HOSTS = "server1.example.com"
$env:REMOTE_USER = "svc_splunk"
$env:REMOTE_PASSWORD = "super-secret"
ansible-playbook -i inventory/hosts.yml cleanup_root_ssh.yml
```

Recommended lifecycle:

```text
1. The runner generates a temporary ed25519 keypair automatically when the initial login user is not root
2. The public key is added to /root/.ssh/authorized_keys using the initial login and sudo su -
3. The main install flow reconnects as root using the generated private key
4. The public key is removed from the remote host after the run
5. The generated private key is deleted from the runner after the run
```

## Security Notes

- Vault tokens, passwords, and retrieved SSH keys are handled with `no_log: true`.
- SSH private key material from Vault is written to a temporary controller file only for the current run.
- Static platform settings stay in [group_vars/all.yml](D:/GitHub%20Projects/Splunk%20Universal%20Forwarder/group_vars/all.yml) and should be moved to protected environment-specific config as needed.
- When `REMOTE_USER=root`, the main UF automation skips bootstrap and connects directly as root.
