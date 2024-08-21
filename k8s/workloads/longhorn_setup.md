# How to install and setup longhorn
Refer to the [official Doc's](https://longhorn.io/docs/1.6.2/) for more information.

Check the status of your Cluster by running this script:
```
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.6.2/scripts/environment_check.sh | bash
```

## Requirements

1. iscsi
    - install it
        ```
        sudo apt install open-iscsi
        ```
    - make sure `iscsi_tcp` is running
        ```
        sudo lsmod | grep iscsi_tcp
        ```
    - or start it with
        ```
        sudo modprobe iscsi_tcp
        ```
2. NSFv4 Client is installed
    - check if the kernel supports NSFv4
        ```
        cat /boot/config-`uname -r`| grep CONFIG_NFS_V4_1
        ```
        ```
        cat /boot/config-`uname -r`| grep CONFIG_NFS_V4_2
        ```
    - install it
        ```
        sudo apt install nfs-common
        ```
3. required programms
    - bash
    - curl
    - findmnt
    - grep
    - awk
    - blkid
    - lsblk
4. `cryptsetup` needs to be installed and `dm_crypt` needs to be loaded on the workers
    - install it
        ```
        sudo apt install cryptsetup
        ```
    - load module with
        ```
        sudo modprobe dm_crypt
        ```
5. persistently load kernel modules
    - worker nodes:
        ```
        cat <<EOF | sudo tee -a /etc/modules-load.d/k8s.conf
        iscsi_tcp
        dm_crypt
        EOF
        ```
    - master nodes:
        ```
        cat <<EOF | sudo tee -a /etc/modules-load.d/k8s.conf
        iscsi_tcp
        EOF
        ```

---

## Installing longhorn with helm
1. Add the Repository and update it
    ```
    helm repo add longhorn https://charts.longhorn.io
    ```
    ```
    helm repo update
    ```
2. Check the version
    ```
    helm search repo longhorn --versions
    ```
3. Install it
    ```
    helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version <VERSION>
    ```
4. Confirm
    ```
    kubectl -n longhorn-system get pod
    ```
    should look like this:
    ```
    NAME                                                READY   STATUS    RESTARTS   AGE
    longhorn-ui-b7c844b49-w25g5                         1/1     Running   0          2m41s
    longhorn-manager-pzgsp                              1/1     Running   0          2m41s
    longhorn-driver-deployer-6bd59c9f76-lqczw           1/1     Running   0          2m41s
    longhorn-csi-plugin-mbwqz                           2/2     Running   0          100s
    csi-snapshotter-588457fcdf-22bqp                    1/1     Running   0          100s
    csi-snapshotter-588457fcdf-2wd6g                    1/1     Running   0          100s
    csi-provisioner-869bdc4b79-mzrwf                    1/1     Running   0          101s
    csi-provisioner-869bdc4b79-klgfm                    1/1     Running   0          101s
    csi-resizer-6d8cf5f99f-fd2ck                        1/1     Running   0          101s
    csi-provisioner-869bdc4b79-j46rx                    1/1     Running   0          101s
    csi-snapshotter-588457fcdf-bvjdt                    1/1     Running   0          100s
    csi-resizer-6d8cf5f99f-68cw7                        1/1     Running   0          101s
    csi-attacher-7bf4b7f996-df8v6                       1/1     Running   0          101s
    csi-attacher-7bf4b7f996-g9cwc                       1/1     Running   0          101s
    csi-attacher-7bf4b7f996-8l9sw                       1/1     Running   0          101s
    csi-resizer-6d8cf5f99f-smdjw                        1/1     Running   0          101s
    instance-manager-b34d5db1fe1e2d52bcfb308be3166cfc   1/1     Running   0          114s
    engine-image-ei-df38d2e5-cv6nc                      1/1     Running   0          114s
    ```

---

## Making the WebUI Accessable through nginx-ingress
1. Create basic auth file
    > It’s important the file generated is named auth (actually - that the secret has a key data.auth)

    ```
    USER=<USERNAME_HERE>; PASSWORD=<PASSWORD_HERE>; echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> auth
    ```
2. Create a secret
    ```
    kubectl -n longhorn-system create secret generic basic-auth --from-file=auth
    ```
3. Create an Ingress manifest `longhorn-ingress.yml`
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: longhorn-ingress
      namespace: longhorn-system
      annotations:
        # type of authentication
        nginx.ingress.kubernetes.io/auth-type: basic
        # prevent the controller from redirecting (308) to HTTPS
        nginx.ingress.kubernetes.io/ssl-redirect: 'false'
        # name of the secret that contains the user/password definitions
        nginx.ingress.kubernetes.io/auth-secret: basic-auth
        # message to display with an appropriate context why the authentication is required
        nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required '
        # custom max body size for file uploading like backing image uploading
        nginx.ingress.kubernetes.io/proxy-body-size: 10000m
    spec:
      ingressClassName: nginx
      rules:
      - host: "<domain>"
        http:
          paths:
            - pathType: Prefix
              path: "/"
              backend:
                service:
                  name: longhorn-frontend
                  port:
                    number: 80
    ```
    or when using tls

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: longhorn-ingress
      namespace: longhorn-system
      annotations:
        # type of authentication
        nginx.ingress.kubernetes.io/auth-type: basic
        # prevent the controller from redirecting (308) to HTTPS
        nginx.ingress.kubernetes.io/ssl-redirect: 'true'
        # name of the secret that contains the user/password definitions
        nginx.ingress.kubernetes.io/auth-secret: basic-auth
        # message to display with an appropriate context why the authentication is required
        nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required '
        # custom max body size for file uploading like backing image uploading
        nginx.ingress.kubernetes.io/proxy-body-size: 10000m
    spec:
      ingressClassName: nginx
      rules:
        - host: "<DOMAIN>"
          http:
            paths:
              - pathType: Prefix
                path: "/"
                backend:
                  service:
                    name: longhorn-frontend
                    port:
                      number: 80
      tls:
        - hosts:
          - <DOMAIN>
    ```
4. apply the manifest
    ```
    kubectl -n longhorn-system apply -f longhorn-ingress.yaml
    ```

---

## Creating a StorageClass
For more Details refer to the [official Docs](https://longhorn.io/docs/1.6.2/nodes-and-volumes/volumes/create-volumes/)

> While installing Longhorn will create a "longhorn" StorageClass.
> In this StorageClass Replicas are set to "3".
> Since I only have 2 Worker-Nodes this will throw an error becaus longhorn cant create a 3rd Replica.

1. Create a StorageClass Manifest and apply it
    ```yaml
    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
      name: longhorn-test
    provisioner: driver.longhorn.io
    allowVolumeExpansion: true
    parameters:
      numberOfReplicas: "2"
      staleReplicaTimeout: "2880" # 48 hours in minutes
      fromBackup: ""
      fsType: "ext4"
    #  mkfsParams: "-I 256 -b 4096 -O ^metadata_csum,^64bit"
    #  diskSelector: "ssd,fast"
    #  nodeSelector: "storage,fast"
    #  recurringJobSelector: '[
    #   {
    #     "name":"snap",
    #     "isGroup":true,
    #   },
    #   {
    #     "name":"backup",
    #     "isGroup":false,
    #   }
    #  ]'
    ```
2. Create a Pod that uses Longhorn Volumes and a corresponding PVC
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: longhorn-volv-pvc
      namespace: default
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn-test
      resources:
        requests:
          storage: 2Gi
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: volume-test
      namespace: default
    spec:
      restartPolicy: Always
      containers:
      - name: volume-test
        image: nginx:stable-alpine
        imagePullPolicy: IfNotPresent
        livenessProbe:
          exec:
            command:
              - ls
              - /data/lost+found
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: volv
          mountPath: /data
        ports:
        - containerPort: 80
      volumes:
      - name: volv
        persistentVolumeClaim:
          claimName: longhorn-volv-pvc
    ```
