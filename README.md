# esx-router

## Dependencies

The `chrony.conf.j2` (and probably other) template uses Ansible's `ipaddr` filter, which requires
the `netaddr` Python library on the **control host** (the machine running
Ansible).

It also playbook uses modules from `community.general` (e.g. `nmcli`). Install the
collection on the control host:

```bash
python3 -m pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml
```

## Topology
```
                                 Internet
                                    |
                                  ens192
                                    |
                            +---------------+
                            |  esx1-router  |
                            +---------------+
                                 |  |  |
                +----------------+  |  +---------------+
                |                   |                  |
             ens256               ens224             ens161
                |                   |                  |
          10.11.0.0/24          10.0.0.0/24      10.11.128.0/24
                                    |
                +-------------------+------------------+
                |                                      |
              ens224                                 ens224
                |                                      |
         +------------------+                  +------------------+
         |    esx2-router   |                  |    esx3-router   |
         +------------------+                  +------------------+
           |             |                       |              |
         ens256        ens161                  ens256         ens161
           |             |                       |              |
     10.12.0.0/24  10.12.128.0/24          10.13.0.0/24   10.13.128.0/24   

```


## Services & responsibilities

- **esx1-router**
  - DNS: `dnsmasq` authoritative for internal zones
  - DHCP: `dnsmasq` for all routed subnets (via relay)
  - NTP: `chronyd`
  - Squid: for isolated/routed clients
  - NAT: masquerade on `internet` zone
- **esx2-router / esx3-router**
  - DNS: `dnsmasq` cache/forwarder → `10.0.0.1`
  - DHCP: `dhcrelay` → `10.0.0.1`
  - NTP: `chronyd`
  - Squid: for isolated/routed clients

## Zones & policies

- `public`: routed networks (ens224/ens256)
- `isolated`: isolated networks (ens161)
- policy `block-to-isolated`: blocks traffic **from** `public/internet` **to** `isolated`
