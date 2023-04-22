# VyOS Ansible Roles

> **Warning**: These roles are not suitable for production use as they may result in lockouts in certain conditions. Additionally, these roles are not perfect and delete functionality may not work as expected. 

This VyOS Ansible Collection is an enhancement of the `vyos.vyos` module and is designed for use with VyOS 1.3.

## Facade Interfaces

A facade interface is an interface that will not be touched by these roles but still needs to be referenced. An example is the interface (most likely `eth0`) that provides internet access.

## Roles

This collection contains the following roles:

- `vyos_firewall`: This role creates/deletes rule sets and zones, and applies zone policies to VLAN, WireGuard, and facade interfaces. It also creates a network group containing all IPv4 CIDR ranges for internal networks, and sets the state policy to sensible values (`established` -> `accept`, `related` -> `accept`, and `invalid` -> `drop`). The role only allows ICMP requests from the network of the interface, and blocks all other traffic by default. 
    - Todos: Allow users to add network groups, add support for additional interface types, currently each interface has its own zone and it should be made so that a zone can contain multiple interfaces.
- `vyos_interfaces`: This role creates/deletes interfaces (only VLAN, WireGuard), configures DHCP, source NAT rules (only for VLAN), destination NAT rules, and WireGuard.
    - Todos: Destination NAT rules cannot be removed because there is no identification field.
- `vyos_setup`: This role disables SSH password authentication, disables the pre-login banner, enables a custom post-login banner, sets the hostname to the inventory name, and sets the nameserver to 8.8.8.8.
    - Todos: Add support for additional nameservers.

