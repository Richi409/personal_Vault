# Wake on Lan

- Install package `ethtool`
- edit the file `/etc/network/interfaces`
    - add `ethernet-wol g` to the interface
    - if this does not work
        - add `post-up ethtool -s enp5s0 wol g` to the interface
- restart the `networking` service
    - `sudo systemctl restart networking`
- show status of interface
    - `sudo ethtool <interface>
