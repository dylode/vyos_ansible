# My Ansible Roles for [VyOS](https://vyos.io/)

> **Warning**: These roles are not suitable for production use as they may result in lockouts in certain conditions. Additionally, these roles are not perfect and not all functionality may not work as expected. 

This repository contains Ansible roles that I use to configure my VyOS routers. These roles use the [vyos.Vyos](https://docs.ansible.com/ansible/latest/collections/vyos/vyos/index.html) Ansible plugin and are designed for use with [VyOS 1.3](https://docs.vyos.io/en/equuleus/).

## Facade Interfaces

A facade interface is an interface that will not be touched by these roles but still needs to be referenced. An example is the interface (most likely `eth0`) that provides internet access.

## Roles

This collection contains the following roles:

- `vyos_firewall`: This role creates/deletes rule sets and zones, and applies zone policies to VLAN, WireGuard, and facade interfaces. It also creates a network group containing all IPv4 CIDR ranges for internal networks, and sets the state policy to sensible values (`established` -> `accept`, `related` -> `accept`, and `invalid` -> `drop`). The role only allows ICMP requests from the network of the interface, and blocks all other traffic by default. 
    - Todos: 
    	- Allow users to add network groups
    	- Add support for additional interface types
    	- Currently each interface has its own zone and it should be made so that a zone can contain multiple interfaces.
- `vyos_interfaces`: This role creates/deletes interfaces (only VLAN, WireGuard), configures DHCP, source NAT rules (only for VLAN) and destination NAT rules (portforwarding).
    - Todos:
    	- Destination NAT rules cannot be removed because there is no identification field.
- `vyos_setup`: This role disables SSH password authentication, disables the pre-login banner, enables a custom post-login banner, sets the hostname to the inventory name, and sets the nameservers.

## Example configuration

```yaml
---
ssh_port: 2222
wireguard_port: 4289

domain: github.com
banner: "Welcome to my VyOS router!"
name_servers:
  - 8.8.8.8
  - 8.8.4.4

interfaces:
  - name: wan
    state: present
    type: facade
    interface: eth0
    dest_nat:
      - state: present
        protocol: tcp
        port: 8080
        target: 10.187.80.4
        target_port: 80

  - name: trunk0
    state: present
    type: facade
    interface: eth1

  - name: finance
    state: present
    type: vlan
    vlan_id: 10
    trunk_port: eth1
    address: 10.187.10.1/24
    source_nat:
      state: present
      outbound_port: eth0
    dhcp:
      state: present
      start: 10.187.10.200
      end: 10.187.10.240

  - name: r&d
    state: present
    type: vlan
    vlan_id: 15
    trunk_port: eth1
    address: 10.187.15.1/24
    source_nat:
      state: present
      outbound_port: eth0
    dhcp:
      state: present
      start: 10.187.15.200
      end: 10.187.15.240

  - name: management
    state: present
    type: vlan
    vlan_id: 20
    trunk_port: eth1
    address: 10.187.20.1/24
    source_nat:
      state: present
      outbound_port: eth0
    dhcp:
      state: present
      start: 10.187.20.200
      end: 10.187.20.240

  - name: wg0
    state: present
    type: wireguard
    address: 10.10.10.1/24
    port: "{{wireguard_port}}"
    peers:
      - name: user1
        state: present
        pubkey: QfJs+TWmNFstPjd9DohSw/xKTKyyqwqKiUASVN2A12Q=
        address: 10.10.10.10/32
     - name: user2
        state: present
        pubkey: QfJs+TWmNFstPjd9DohSw/xKTKyyqwqKiUASVN2A12Q=
        address: 10.10.10.11/32   
     - name: user3
        state: present
        pubkey: QfJs+TWmNFstPjd9DohSw/xKTKyyqwqKiUASVN2A12Q=
        address: 10.10.10.12/32   

firewall:
  rulesets:
    - name: allow-from-wan
      default_action: drop
      rules:
        - number: 10
          description: "ssh rate limit"
          action: drop
          protocol: tcp
          recent:
            count: 4
            time: 60
          state:
            new: true
          destination:
            port: "{{ssh_port}}"

        - number: 20
          description: "allow ssh"
          action: accept
          protocol: tcp
          state:
            new: true
          destination:
            port: "{{ssh_port}}"

        - number: 30
          description: "allow wireguard"
          action: accept
          protocol: udp
          destination:
            port: "{{wireguard_port}}"

    - name: allow-web
      default_action: drop
      rules:
        - number: 10
          action: accept
          protocol: tcp
          destination:
            port: 80

    - name: allow-internet
      default_action: accept
      rules:
        - number: 10
          description: "deny to internal networks"
          action: drop
          destination:
            group:
              network_group: "InternalNetworks"

    - name: allow-wg-to-internal
      default_action: drop
      rules:
        - number: 10
          description: "allow wg0 to InternalNetworks"
          action: accept
          destination:
            group:
              network_group: "InternalNetworks"
       
  zone_policies:
    - from: wan
      to: local
      policy: allow-from-wan
    - from: 
        - finance
        - r&d
        - management
      to: wan
      policy: allow-internet
    - from: wg0
      to:
        - finance
        - r&d
      policy: allow-wg-to-internal
    - from: wan
      to: management
      policy: allow-web

```

