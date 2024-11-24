# Simple Router Setup

## DHCP Server
1. install packages
    ```bash
    sudo apt install isc-dhcp-server
    ```
2. configuration
    - file `/etc/defaults/isc-dhcp-server`
    ```
    DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
    #DHCPDv6_CONF=/etc/dhcp/dhcpd6.conf
    DHCPDv4_PID=/var/run/dhcpd.pid
    #DHCPDv6_PID=/var/run/dhcpd6.pid
    #OPTIONS=""
    INTERFACESv4="ens19 ens20"
    # INTERFACESv6=""
    ```
    - file `etc/dhcp/dhcpd.conf`
    ```
    option domain-name "rm-hlab.com";
    option domain-name-servers 192.168.2.11;
    
    default-lease-time 600;
    max-lease-time 7200;
    ddns-update-style none;
    
    authoritative;
    
    #log-facility local7;
    
    subnet 192.168.3.0 netmask 255.255.255.0 {
      range 192.168.3.20 192.168.3.100;
      option domain-name-servers 192.168.2.11;
      option domain-name "rm-hlab.com";
      option routers 192.168.3.1;
      option broadcast-address 192.168.3.255;
      default-lease-time 600;
      max-lease-time 7200;
    }
    
    subnet 192.168.4.0 netmask 255.255.255.0 {
      range 192.168.4.20 192.168.4.100;
      option domain-name-servers 192.168.2.11;
      option domain-name "rm-hlab.com";
      option routers 192.168.4.1;
      option broadcast-address 192.168.4.255;
      default-lease-time 600;
      max-lease-time 7200;
    }
    ```
3. enable IPv4 Forwarding
    - uncomment `net.ipv4.ip_forward=1` in the file `/etc/sysctl.conf`


# Test Setup

## Cloud-init
```yaml
#cloud-config

# Set hostname (user can provide via Proxmox UI or metadata)
hostname: <hostname>
manage_etc_hosts: true

# instsall needed packages
packages:
  - sudo

# Configure SSH access with public key (user-provided SSH key)
ssh_pwauth: false  # Disable password-based SSH login
users:
  - name: ansible
    gecos: Ansible User
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    lock_passwd: true  # Disable password login for this user
    shell: /bin/bash
    ssh_authorized_keys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKQKHTAsdLbHm//BhhvwJVF5QN2nkB9C6qMwDPepfMat richard-tuxedo"  # User-provided SSH public key
  - name: test
    gecos: testuser
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    lock_passwd: false
    passwd: $6$rounds=4096$E9Fdl8MKzmjfyJY4$biyaFAIDGB1Oe4wMc5XykbYakrfphKGRGCaaNxlrwrgynxckvCEBxWcYUkC4FUyKH8yMbxqtMwwFkgHrFV2ax0
    shell: /bin/bash
    ssh_authorized_keys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKQKHTAsdLbHm//BhhvwJVF5QN2nkB9C6qMwDPepfMat richard-tuxedo"  # User-provided SSH public key

# disable ssh root login
disable_root: true
# disable ssh password authentication
ssh_pwauth: false
```
- save under `/var/lib/vz/snippets/`

## VM config
- clone the template
- add cloud-init drive
- specify the `user-data.yaml` file to be used
    ```bash
    qm set <vmid> --cicustom "user=local:snippets/user-data-<vmid>.yaml"
    ```
    ```bash
    qm set 110 --cicustom "user=local:snippets/user-data-110.yaml"
    qm set 111 --cicustom "user=local:snippets/user-data-111.yaml"
    qm set 112 --cicustom "user=local:snippets/user-data-112.yaml"
    ```
- specify the ip addresses
    ```bash
    qm set <vmid> -ipconfig0 ip=<ip>/<mask>,gw=<gw-ip>
    ```
    ```bash
    qm set 110 -ipconfig0 ip=192.168.2.40/24,gw=192.168.2.1
    qm set 111 -ipconfig0 ip=192.168.2.41/24,gw=192.168.2.1
    qm set 112 -ipconfig0 ip=192.168.2.42/24,gw=192.168.2.1
    ```
- specify dns server
    ```bash
    qm set <vmid> --nameserver "<dns-ip>"
    ```
    ```bash
    qm set 110 --nameserver "192.168.2.11"
    qm set 111 --nameserver "192.168.2.11"
    qm set 112 --nameserver "192.168.2.11"
    ```
