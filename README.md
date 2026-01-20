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
  - LB: `haproxy` for OpenShift API/Ingress when `lb.managed: user`
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

## Cluster definitions

Clusters are defined in `group_vars`/`host_vars` under `clusters`. The playbook derives DNS/DHCP
entries and (when `lbManaged: user`) HAProxy VIPs, and also writes the ABI inventory to
`inventoryBaseDir/<cluster name>.yaml` when it's requested.

Examples:
```yaml
clusters:
  - name: ocp4-simple-dynamic
    type: simple
    cluster_definition:
      baseDomain: gmbros.local

      controlPlaneReplicas: 3
      computeReplicas: 2

      machineNetwork:
        cidr: 10.11.0.0/24

      lbManaged: cluster
      ingressVIP: 10.11.0.100
      apiVIP: 10.11.0.101
```

```yaml
clusters:
  - name: ocp4-abi-vsphere-ha-dynamic
    type: abi-vsphere
    cluster_definition:
      inventoryBaseDir: /home/sgarcia/src/ocp4-abi-vsphere-ansible/inventories
      clusterBaseDir: ~/.local/ocp4/clusters

      baseDomain: gmbros.local

      controlPlaneReplicas: 3
      computeReplicas: 2

      networkType: OVNKubernetes
      machineNetwork:
        cidr: 10.11.0.0/24

      rendezvousIP: 10.11.0.103

      lbManaged: cluster
      ingressVIP: 10.11.0.100
      apiVIP: 10.11.0.101

      vms:
        controlPlaneData:
          cluster: zone1
          cpu: 4
          memory: 16384
          disk:
          - 100
          networks:
          - name: ens192
            network: VM Routed Network
          datastore: esx1-datastore
        computeData:
          cluster: zone1
          cpu: 4
          memory: 8192
          disk:
          - 100
          networks:
          - name: ens192
            network: VM Routed Network
          datastore: esx1-datastore
```

Notes:
- If `lbManaged: cluster`, only DNS for `api`/`api-int` is created; no HAProxy.
- The first 3 node IPs are used as API backends; ingress uses the remaining nodes (or the same 3
  if only 3 nodes exist).

## Interface cleanup

Set `interfaces_force_cleanup: true` to reset each interface to only its base address from the
`interfaces` list. The same cleanup runs when HAProxy is disabled (see `lb_cleanup_on_disable`).
