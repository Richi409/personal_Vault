# How to create a LVM

## Create a Partition with Type `Linux LVM`
- example with `fdisk`
- `sudo fdisk /dev/<blk-device>`
- `g` $\Rightarrow$ create new GPT Partition table (only if you dont care about the contents of the harddrive)
- `n` $\Rightarrow$ create new Partition
    - make it as big as you want
- `t` $\Rightarrow$ change Partition-Type
    - make sure to select the correct partition if you have more than one
    - select Partition Type `43` for Linux LVM
- `w` $\Rightarrow$ write changes to disk

## Creating the LVM
1. Creating Persistant Volume (PV)
    ```bash
    sudo pvcreate /dev/<blk-device><partition_number>
    ```
    - list all the existing PV's
        ```bash
        sudo pvs
        ```
2. Creating a Volume Group (VG)
    ```bash
    sudo vgcreate <name> /dev/<blk-device><partition_number> [<blk-device><partition_number> ...]
    ```
    - list existing VG's
        ```bash
        sudo pvdisplay
        ```
        ```bash
        sudo vgdisplay
        ```
3. Create a Logical Volume (LV)
    - specific size
    ```bash
    sudo lvcreate -n <name> -L<size>G <vg-name>
    ```
    - percent of available space in VG
    ```bash
    sudo lvcreate -n <name> -l<percent>%VG <vg-name>
    ```
    - percent of free space in VG
    ```bash
    sudo lvcreate -n <name> -l<percent>%FREE <vg-name>
    ```
    - list existing VG's
        ```bash
        sudo lvdisplay
        ```
4. Create filesystem on LV
    ```bash
    sudo mkfs.ext4 /dev/<vg-name>/<lv-name>
    ```
5. make entry in fstab
    ```bash
    /dev/mapper/<vg-name>-<lv-name> </path/to/mountpoin>               ext4    errors=remount-ro 0       1
    ```
    - reload config
        ```bash
        sudo systemctl daemon-reload
        ```
    - apply fstab configuration
        ```bash
        sudo mount -a
        ```
