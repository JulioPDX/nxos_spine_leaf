# Ansible NXOS Spine Leaf Infrastructure

![](./images/spine_leaf.PNG)

## Purpose

This repo is based on the great post by [Rick Donato](https://www.packetflow.co.uk/how-to-build-a-nxos-9000v-based-evpn-vxlan-fabric/). I wanted to see if I could automate this guide using Ansible. Ansible is a popular configuration management framework that probably doesn't need much more of an introduction. In the repo you will see a decent amount of use cases. Whether thats using built in modules for specific tasks, jinja templating (building configurations), conditionals, and variable placements.

## Prerequisites

These are just what is running in my local environment!
1. Ansible >= 2.9
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

This file will kick off each role listed. Roles are broken out by function, again going with what made sense to me. These could probably be split off even more for more organization. 

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

My original thought with this repo was to use as many native modules that came with Ansible. For example, if there was a module only to configure a hostname for nxos devices then I would use that. Eventually I found it easier and more flexible to just make a jinja template and run with the nxos_config module. More on that later.

```yaml
- name: configure necessities
  nxos_config:
    lines:
      - hostname {{ hostname }}
      - cli alias name wr copy run start
    save_when: modified
  when: hostname is defined
... snip
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
```

When that task is run against leaf1, the final output will be `hostname leaf1` ... 
