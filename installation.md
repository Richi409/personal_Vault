# Installation of Kubernets on Debian 12

## Preparation of Debian 12
- make sure swap is disabled
    - `swapoff -a`
    - edit the `/etc/fstab` to make the change persistent
- make sure you have a user with sudo rights (not root user)
- make sure the system is up to date with `sudo apt update && sudo apt upgrade`

## Installation and Configuration of Container Runtime (containerd)
- install containerd 
    ```
    sudo apt install containerd
    ```
- generate the default containerd cofniguration file
    ```
    containerd config default | sudo tee /etc/containerd/config.toml
    ```
- edit the /etc/containerd/config.toml
    - under `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]` set the option `SystemdCgroup = true`

---

## Enable IPv4 packet forwarding
- run 
    ```
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.ipv4.ip_forward = 1
    EOF
    ```
- apply settings with 
    ```
    sudo sysctl --system
    ```
- verify settings with
    ```
    sudo sysctl net.ipv4.ip_forward
    ```

---

## Specifying Kernel Modules that need to be loaded
- run
    ```
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf 
    overlay 
    br_netfilter
    EOF
    ```
- :warning: reboot the machine for the changes to take effect

---

## Installing kubernetes tools

- install dependencies
    ```
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg
    ```
- add gpg key
    - make sure the `keyrings` folder existes (create it with `mkdir -p -m 755 /etc/apt/keyrings` if necessary)
    ```
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```
- add the kubernetes apt repository
    ```
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```
    - make sure to adjust the version if needed
- load the new source with `sudo apt update`
- install the needed packages
    ```
    sudo apt install -y kubelet kubeadm kubectl
    ```
- mark the packages as hold
    - packages wont get updated automatically
        ```
        sudo apt-mark hold kubelet kubeadm kubectl
        ```
    - show packages that are currently on hold
        ```
        sudo apt-mark showhold
        ```
    - hold can be removed with
        ```
        sudo apt-mark unhold kubelet kubeadm kubectl
        ```
- start process and 
    ```
    sudo systemctl enable --now kubelet
    ```

---

## Creating Kubernetes Cluster

> :heavy_exclamation_mark: only run this on the master node
- Initializing Kubernetes Cluster (Single Master Node)
    ```
    sudo kubeadm init --control-plane-endpoint=<ip-of control-plane node> --node-name <control-plane node host-name> --pod-network-cidr=10.244.0.0/16
    ```
- Initializing Kubernetes Cluster (Multiple Master Nodes)
    ```
    sudo kubeadm init --control-plane-endpoint=<ip-of control-plane node> \
    --node-name <control-plane node host-name> \
    --pod-network-cidr=10.244.0.0/16 \
    --upload-certs
    ```
- run 
    - `mkdir -p $HOME/.kube`
    - `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`
    - `sudo chown $(id -u):$(id -g) $HOME/.kube/config`
- install overlay network
    ```
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
    ```
    or
    ```
    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
    ```
    - coredns pods should now be running


### Adding a Node to the Cluster
> :heavy_exclamation_mark: Do not run on the master node

While initializing the cluster a "join-command" should have been printed to stdout. </br> To add a Node just execute this command with **sudo** on it.

- the join command has a timelimit, so after some time you have to regenerate this join command (on the master node)
    ```
    sudo kubeadm token create --print-join-command
    ```
