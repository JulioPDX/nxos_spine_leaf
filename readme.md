# Ansible NXOS Spine Leaf Infrastructure

![](./images/spine_leaf.PNG)

## Purpose

This repo is based on the great post by [Rick Donato](https://www.packetflow.co.uk/how-to-build-a-nxos-9000v-based-evpn-vxlan-fabric/). I wanted to see if I could automate this guide using Ansible. Ansible is a popular configuration management framework that probably doesn't need much more of an introduction. In the repo you will see a decent amount of use cases. Whether thats using built in modules for specific tasks, jinja templating (building configurations), conditionals, and variable placements.

## Prerequisites

These are just what is running in my local environment!
1. Ansible =< 2.9
2. NXOSv 9K Version 9.2.4
3. Validate Ansible control machine has access to nodes

## Project Structure

```
juliopdx@librenms:~/repos$ tree nxos_spine_leaf/
nxos_spine_leaf/
├── ansible.cfg
├── deploy.yaml
├── group_vars
│   ├── leafs.yaml
│   ├── network.yaml
│   └── spines.yaml
├── hosts
├── host_vars
│   ├── leaf1.yaml
│   ├── leaf2.yaml
│   ├── leaf3.yaml
│   ├── leaf4.yaml
│   ├── spine1.yaml
│   └── spine2.yaml
├── images
│   └── spine_leaf.PNG
├── readme.md
└── roles
    ├── base
    │   └── tasks
    │       └── main.yaml
    ├── bgp
    │   ├── files
    │   ├── tasks
    │   │   └── main.yaml
    │   └── templates
    │       └── bgp.j2
    ├── enhancements
    │   └── tasks
    │       └── main.yaml
    ├── features
    │   └── tasks
    │       └── main.yaml
    ├── interfaces
    │   └── tasks
    │       └── main.yaml
    ├── ospf
    │   └── tasks
    │       └── main.yaml
    ├── pim
    │   └── tasks
    │       └── main.yaml
    └── vlan_vxlan
        ├── files
        ├── tasks
        │   └── main.yaml
        └── templates
            └── vlan_vxlan.j2
```

Each host has an individual file under the `host_vars` folder and each group has an individual file under the `group_vars` folder. This is again for organization. If a variable is used throughout all devices, then it will be placed in `network.yaml` file. If a variable is specific to the leafs then it will be placed in the `leafs.yaml` file. lastly, if a variable is host specific, it will be under the `host_vars` folder for each host.

### `hosts`

I tried to make the hosts file as basic as possible, grouping together nodes where it made sense to me.

```
locahost

[network]
spine1 ansible_host=192.168.10.83
spine2 ansible_host=192.168.10.84
leaf1 ansible_host=192.168.10.90
leaf2 ansible_host=192.168.10.86
leaf3 ansible_host=192.168.10.87
leaf4 ansible_host=192.168.10.88

[network:vars]
ansible_user=cisco
ansible_ssh_pass=cisco
ansible_network_os=nxos

[spines]
spine1
spine2

[leafs]
leaf1
leaf2
leaf3
leaf4
```

Not much to look at in the ansible.cfg file, these are just normal settings with some timers set to account for any latency. After the hosts file and config file are created, we can get started on the fun stuff. 

### `deploy.yaml`

This file will kick off each role listed. Roles are broken out by function, again going with what made sense to me. These could probably be split off even more for organization. 

```yaml
---
- name: Deploying Infrastructure
  connection: network_cli
  gather_facts: false
  hosts: network
  roles:
    - base
    - features
    - interfaces
    - ospf
    - pim
    - bgp
    - enhancements
    - vlan_vxlan
```

### `roles/base/tasks/main.yaml`

My original thought with this repo was to use as many native nxos modules that come with Ansible. For example, if there was a module only to configure a hostname for nxos devices then I would use that. Eventually I found it easier and more flexible to just make a jinja template and run with the nxos_config module. More on that later.

```yaml
- name: configure necessities
  nxos_config:
    lines:
      - hostname {{ hostname }}
      - cli alias name wr copy run start
    save_when: modified
  when: hostname is defined

--snipp--
```

You will notice this is the first time you see a variable that wil be populated, `hostname {{  hostname }}`. Since this is device specific, here is a snip from `leaf1.yaml`.

```yaml
hostname: leaf1

interfaces:

  Ethernet1/1:
    name: Ethernet1/1
    enabled: True
    mode: layer3
    mtu: 9216
    description: connection to spine1
    ospf_enabled: True

--snipp--
```

When that task is run against leaf1, the final output will be `hostname leaf1` ... 

### `roles/features/tasks/main.yaml`

```yaml
- name: enable features for fabric
  nxos_feature:
    feature: "{{ item }}"
    state: enabled
  loop: "{{ features }}"
  when: features is defined

- name: enable additional features without using nxos_feature module
  nxos_config:
    lines: ['feature fabric forwarding', 'nv overlay evpn']
```

This file is a pretty cool example on how to solve the same task. The first task uses the `nxos_feature` module and loops over the `features` variable. Since all features are the same for each node, they are located under `group_vars/network.yaml`. The second option was to use the `nxos_config` module and passing in a list.

```yaml
ospf_instance: OSPF_UNDERLAY_NET

features:
  - ospf
  - pim
  - bgp
  - interface-vlan
  - vn-segment-vlan-based
  - nv overlay

--snipp--
```

### `roles/interfaces/tasks/main.yaml`

```yaml
- name: configure ethernet interface settings
  nxos_interfaces:
    config:
      - name: "{{ item.value.name }}"
        description: "{{ item.value.description }}"
        enabled: "{{ item.value.enabled }}"
        mtu: "{{ item.value.mtu }}"
        mode: "{{ item.value.mode }}"
  with_dict: "{{ interfaces }}"
  when: interfaces is defined

--snipp--
```

As you can see this is usable but a bit messy. In this case we are looping over a dict that we've defined as `interfaces` for each host. Remember the `leaf1.yaml` snip shown earlier.

### `roles/ospf/tasks/main.yaml`

```yaml
--snip--
- name: enabling ospf on interfaces
  nxos_config:
    lines:
      - medium p2p
      - ip unnumbered loopback0
      - ip router ospf {{ ospf_instance }} area 0.0.0.0
    parents: interface {{ item.value.name }}
  with_dict: "{{ interfaces }}"
  when: interfaces is defined

- name: enabling ospf on loopbacks
  nxos_interface_ospf:
    interface: "{{ item.value.name }}"
    ospf: "{{ ospf_instance }}"
    area: '0.0.0.0'
  with_dict: "{{ loopbacks }}"
  when: loopbacks is defined
```

Similar to before, two different approaches to enable OSPF on an interface. Ill move into the bgp and vlan_vxlan roles as they both utilize jinja templates.

### `roles/bgp/tasks/main.yaml`

```yaml
- name: generate bgp configurations
  template:
    src: ./roles/bgp/templates/bgp.j2
    dest: ./roles/bgp/files/{{ hostname }}.cfg
  delegate_to: localhost

- name: configure bgp
  nxos_config:
    src: ./roles/bgp/files/{{ hostname }}.cfg
```

Nothing crazy here, we are asking the local Ansible control machine to generate configuration files and then using that file with the `nxos_config` module to send the configuration. Ill include a small section of the `bgp.j2` template used to build the configuration file.

```jinja
router bgp {{ bgp.asn }}
  log-neighbor-changes
  address-family ipv4 unicast
  address-family l2vpn evpn
{% if bgp.spine %}
    retain route-target all
{% endif %}
  template peer VXLAN
    remote-as 64520
    update-source loopback0
    address-family ipv4 unicast
      send-community extended
{% if bgp.spine %}
      route-reflector-client
{% endif %}

--snip--
```

### `roles/vlan_vxlan/tasks/main.yaml`

```yaml
- name: generate vlan/vni/evpn configurations
  template:
    src: ./roles/vlan_vxlan/templates/vlan_vxlan.j2
    dest: ./roles/vlan_vxlan/files/{{ hostname }}.cfg
  delegate_to: localhost
  when: "'leafs' in group_names"

- name: configure vlan/vni/evpn
  nxos_config:
    src: ./roles/vlan_vxlan/files/{{ hostname }}.cfg
  when: "'leafs' in group_names"
```

Same thought as before with the bgp role. Local Ansible to generate configurations and then send. One neat wrinkle here is that these configuration only apply to leafs. So we use a conditional statement to only run on nodes in the leafs group. Heres a small snip of the `vlan_vxlan.j2` template. 

```jinja
{% for vlan in vlans %}
vlan {{ vlan.id }}
  name {{ vlan.description }}
{% if not vlan.enabled %}
  shutdown
{% endif %}
{% if vlan.vn_segment %}
  vn-segment {{ vlan.vn_segment }}
{% endif %}
{% endfor %}

{% for vrf in vrfs %}
{% if vrf.enabled %}
vrf context {{ vrf.name }}
{% if vrf.vn_segment %}
  vni {{ vrf.vn_segment }}
{% if vrf.rd %}
  rd {{ vrf.rd }}
{% if vrf.address_family %}
  {{ vrf.address_family }}
    route-target both auto
    route-target both auto evpn
{% endif %}
{% endif %}
{% endif %}
{% endif %}
{% endfor %}

--snip--
```