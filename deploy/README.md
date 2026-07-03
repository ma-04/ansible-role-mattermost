# Mattermost deployment harness

Runnable playbook for the `ansible-role-mattermost` role (in the repo root).

Setup and full documentation are in the top-level [README](../README.md) —
see [The `deploy/` harness](../README.md#the-deploy-harness).

Quick version:

```bash
cd deploy
cp inventory.ini.example inventory.ini                             && $EDITOR inventory.ini
cp group_vars/chat/vars.yml.example group_vars/chat/vars.yml       && $EDITOR group_vars/chat/vars.yml
cp group_vars/chat/vault.yml.example group_vars/chat/vault.yml     && $EDITOR group_vars/chat/vault.yml
ansible-vault encrypt group_vars/chat/vault.yml
ansible-galaxy collection install -r ../requirements.yml
ansible-playbook site.yml --ask-vault-pass
```
