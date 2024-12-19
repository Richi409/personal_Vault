# Cloud-init Setup

1. Install and Configure VM however you like
    - do not allow login as root
    - create user `ansible`
        - password does not matter
2. Install cloud-init
    ```bash
    apt install cloud-init
    ```
3. Specify datasources
    ```bash
    echo "datasources_list: [ NoCloud, ConfigDrive ]" | sudo tee /etc/cloud/cloud.cfg.d/99_pve.cfg
    ```
4. Enable Services
    ```bash
    systemctl enable cloud-init
    ```
    ```bash
    systemctl enable cloud-init-local
    ```
    ```bash
    systemctl enable cloud-init-network
    ```
    ```bash
    systemctl enable cloud-final
    ```
5. Set Sudo privileges
    ```bash
    echo "ansible ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/ansible
    ```
6. SSHD Config
    - `PasswordAuthentication no`
    - `PermitRootLogin no`
7. Create a rule in `/etc/security/access.conf`
    ```bash
    echo "-:ansible:LOCAL" | sudo tee -a /etc/security/access.conf
    ```
8. Uncomment `account  required  pam_access.so` in `/etc/pam.d/login`
9. Convert from `ifupdown` to `netplan` with `systemd-networkd` as backend
    - cloud-init integrates best with `netplan`, while not working as good with classic `ifupdown` configuration
    - uninstall `ifupdown`
        ```bash
        sudo apt purge ifupdown
        ```
    - enable and start `systemd-networkd` Service
        ```bash
        sudo systemctl enable --now systemd-networkd
        ```
    - install required packages
        ```bash
        sudo apt install netplan.io systemd-resolved
        ```
    - enable and start `systemd-resolved` Service
        ```bash
        sudo systemctl enable --now systemd-resolved
        ```
    - move old network configuration or remove it
        ```bash
        sudo mv /etc/network/interfaces /etc/network/interfaces.bak
        ```
        ```bash
        sudo rm -r /etc/network/interfaces*
        ```
    - remove file `/etc/resolv.conf`
        ```bash
        sudo rm /etc/resolv.conf
        ```
    - Create a Symlink to replace `/etc/resolv.conf`
        ```bash
        sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
        ```
10. Clean System
    ```bash
    rm -rf /etc/ssh/ssh_host_*
    ```
    ```bash
    rm /etc/udev/rules.d/70-persistent-net.rules
    ```
    ```bash
    cloud-init clean
    ```
11. Power Off the System
