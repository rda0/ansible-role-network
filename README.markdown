network
=======

This role configures the network. The following backends are currently supported:

- systemd-networkd: `include_network_systemd: True`

systemd-networkd
----------------

Two variables `network_systemd_network` and `network_systemd_netdev` define the files (`*.network` and `*.netdev`) created in `/etc/systemd/network`. Each variable may contain a list of dictionaries with keys `name` (the filename) and `sections` (the file content). `sections` contains a list of sections to be created in the file. The `params` field contains the list of parameters to be listed in the section named after the value of `section.name`.

The syntax is as follows:

```yaml
network_systemd_network:
  - name: filename
    sections:
    - name: SectionName
      params:
        - Parameter
```

For example to configure all ethernet interfaces using DHCPv4:

```yaml
network_systemd_network:
  - name: en
    sections:
    - name: Match
      params:
        - 'Name=en*'
    - name: Network
      params:
        - 'DHCP=ipv4'
        - 'Domains=phys.ethz.ch ethz.ch'
    - name: DHCP
      params:
        - 'UseDomains=true'
        - 'UseNTP=true'
```

For a detailed example with bonding, vlans and bridges, see `production/host_vars/phd-web/network.yml`

Note: If you are bootstrapping a new host or enabling systemd-networkd for the first time, you should reboot the host after the play has been run. Only interfaces brought up will be configured by systemd-network. Migrating to `/run/systemd/resolve/resolv.conf` as it is recommended and done in this role may break DNS resolution until the host is rebooted.

Note2: You probably should not migrate from another backend without careful testing and manual preparation.
