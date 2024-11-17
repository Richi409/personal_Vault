# Cloud-init Setup

1. Install and Configure VM however you like
2. Install cloud-init
    ```bash
    apt install cloud-init
    ```
3. Specify datasources
    ```bash
    echo "datasources_list: [ NoCloud, ConfigDrive ]" > /etc/cloud/cloud.cfg.d/99_pve.cfg
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
5. Clean System
    ```bash
    rm -rf /etc/ssh/ssh_host_*
    ```
    ```bash
    rm /etc/udev/rules.d/70-persistent-net.rules
    ```
    ```bash
    cloud-init clean
    ```
6. Power Off the System
