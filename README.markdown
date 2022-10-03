network
=======

This role configures the network. The following backends are currently supported:

- systemd-networkd: `import_network_systemd: True`

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

kernel settings
---------------

The role may need to set some kernel parameters (`import_network_kernel_settings`). Some are automatically set and some need to be manually set (see `defaults/main.yml`).

Should a setting change to use the kernel's default setting, either a reboot or manually resetting is needed.

### network_multiple_interfaces_on_same_subnet

If multiple interfaces are connected to the same layer 2 subnet (vlan) and even if no IP address is configured on some interfaces, then kernel settings need tuning to avoid possible network or network monitoring infrastructure problems.
This is automatically tuned for bridge templates, when they are using the variables `network_interface` and `network_bridges` and when the host is configured with a dedicated management interface (assuming bridge and management interface are on the same subnet).
By default the kernel is configured to respond to ARP requests for a given IP address on any interface to increase the chance of successful communication. But from the perspective of a network monitoring infrastructure this may appear as an IP address which is configured on multiple interfaces (MAC addresses).
For such cases set `network_multiple_interfaces_on_same_subnet: True`.

See also:

- https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt
- https://wiki.openvz.org/Multiple_network_interfaces_and_ARP_flux
- https://lwn.net/Articles/45373/

### network_ipv4_ip_forward

Set to `True` to enable IPv4 forwarding to configure the host as a router. This may also be set dynamically by software like libvirt or other hypervisors (container, vms) when needed.

### network_netfilter_nf_conntrack_max

Increase this if the conntrack table is getting full, otherwise connections may get dropped.
