network
=======

This role configures the network. The following backends are currently supported:

- systemd-networkd: `include_network_systemd: True`

systemd-networkd
----------------

Two variables `network_systemd_network` and `network_systemd_netdev` can be used to override the default configuration, which is using preset templates. They define the files (`*.network` and `*.netdev`) created in `/etc/systemd/network`. Each variable may contain a list of dictionaries with keys `name` (the filename) and `sections` (the file content). `sections` contains a list of sections to be created in the file. The `params` field contains the list of parameters to be listed in the section named after the value of `section.name`.

Usually these two variables should not be used in per host configs, as they are preset with templates by default (see below). Instead use the variables `network_systemd_network_local` and `network_systemd_netdev_local`, which are merged with the preset variables.

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

Hosts configured as such with mutliple interfaces need the following setting to speed up the boot process:

```yaml
network_systemd_networkd_wait_online_any: True
```

Another example to configure an interface in addition to preset interfaces:

```yaml
network_systemd_network_local:
  - name: "eth1"
    sections:
    - name: Match
      params: ['Name=eth1']
    - name: Network
      params:
        - 'IPv6AcceptRA=false'
        - 'ConfigureWithoutCarrier=true'
    - name: Address
      params:
        - "Address=192.168.0.42/24"
        - 'Scope=link'
```

Or to create additional interfaces:

```yaml
network_systemd_netdev_local:
  - name: tun0
    sections:
    - name: NetDev
      params:
        - 'Name=tun0'
        - 'Kind=tun'
```

### bootstrapping

Note: If you are bootstrapping a new host or enabling systemd-networkd for the first time, you should reboot the host after the play has been run. Only interfaces brought up will be configured by systemd-network. Migrating to `/run/systemd/resolve/resolv.conf` as it is recommended and done in this role may break DNS resolution until the host is rebooted.

### templates

Some commonly used configurations are available as templates in `vars/main.yml` in the dicts `network_systemd_network_templates` and `network_systemd_netdev_templates`.

Required variables:

- `network_interface`

Optional variables (for bridge templates):

- `network_bond`
- `network_vlans`
- `network_bridges`

Example:

```yaml
network_systemd_template: bond_2vlans_2bridges
network_interface: br-dc2552
network_bond:
  name: bond1
  interfaces: ['eno3', 'eno4']
network_vlans:
  vlans:
    - name: dc2552
      id: 2552
    - name: dc2745
      id: 2745
  interface: bond1
network_bridges:
  - name: br-dc2552
    interface: dc2552
  - name: br-dc2745
    interface: dc2745
```

### defaults

For details see `defaults/main.yml`.

- `network_systemd_clean_install`: By default the role will perform a clean install, removing/disabling any other known default Debian/Ubuntu network configuration
- `network_systemd_wipe`: Should only manually be set to `True` via command line parameter (`-e network_systemd_wipe=True`), when `/etc/systemd/network` is expected to have existing configration files not needed anymore or when `.network` or `.netdev` file names change

### kernel settings

The role may need to set some kernel parameters, this is only active when needed (`include_network_kernel_settings`). It will automatically enable that when a template name contains the string `bridge` and use the approriate settings.

If multiple interfaces are connected to the same layer 2 subnet, some kernel settings need tuning to avoid possible network problems. This is automatically detected for bridge templates, when they are using the variables `network_interface` and `network_bridges`.
