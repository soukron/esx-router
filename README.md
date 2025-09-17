# esx-router

## Dependencies (control host)

The `chrony.conf.j2` (and probably other) template uses Ansible's `ipaddr` filter, which requires
the `netaddr` Python library on the **control host** (the machine running
Ansible).

Install with one of these options:

```bash
sudo dnf install -y python3-netaddr
```

or

```bash
python3 -m pip install -r requirements.txt
```

## Ansible collections

This playbook uses modules from `community.general` (e.g. `nmcli`). Install the
collection on the control host:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```
