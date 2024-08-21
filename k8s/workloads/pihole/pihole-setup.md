# How to setup pihole in Kubernetes
## Deploying pihole using helm
Check out the github [repository](https://github.com/MoJo2600/pihole-kubernetes) for more information

1. Add the repository
    ```
    helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
    helm repo update
    ```
2. create a `values.yaml`
    ```yaml
    serviceDns:
      mixedService: false
      type: LoadBalancer
      port: 53
      externalTrafficPolicy: Cluster
      loadBalancerIP: 192.168.2.51
      annotations:
        metallb.universe.tf/address-pool: <ADDR_POOL_NAME>
        metallb.universe.tf/allow-shared-ip: <SHARED_IP_ANNOTATION_NAME>
    
    serviceDhcp:
      enabled: false
    
    serviceWeb:
      http:
        enabled: true
        port: 80
      https:
        enabled: false
      type: ClusterIP
      externalTrafficPolicy: Local
    
    virtualHost: pi.hole
    
    ingress:
      enabled: true
      ingressClassName: nginx
      path: /
      hosts:
        - <FQDN>
      tls:
        - hosts:
          - <FQDN>
    
    persistentVolumeClaim:
      enabled: true
      accessModes:
        - ReadWriteOnce
      size: "500Mi"
      storageClass: "longhorn"
    
    adminPassword: "admin"
    
    # -- Use an existing secret for the admin password.
    admin:
      # -- If set to false admin password will be disabled, adminPassword specified above and the pre-existing secret (if specified) will be ignored.
      enabled: true
      # -- Specify an existing secret to use as admin password
      existingSecret: ""
      # -- Specify the key inside the secret to use
      passwordKey: "password"
    ```
3. Install pihole
    ```
    helm install pihole mojo2600/pihole --namespace pihole --create-namepsace -f values.yaml
    ```

### Deploying `external-dns` using Helm

1. Add the `bitnami` helm repository
    ```
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update
    ```
2. create a `values.yaml`
    ```yaml
    provider: pihole
    pihole:
      server: "http://pihole-web.pihole.svc.cluster.local"
    extraEnvVars:
      - name: EXTERNAL_DNS_PIHOLE_PASSWORD
        valueFrom:
          secretKeyRef:
            name: pihole-password
            key: password
    ```
3. Install the helm chart
    ```
    helm install external-dns bitnami/external-dns --namespace pihole -f values.yaml
    ```

### Deploying `external-dns` using Yaml-Manifests
external-dns automatically creates A-Records for existing ingress objects.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: pihole
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
  namespace: pihole
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
  namespace: pihole
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: pihole
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: pihole
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.14.2
        # If authentication is disabled and/or you didn't create
        # a secret, you can remove this block.
        # envFrom:
        # - secretRef:
        #     # Change this if you gave the secret a different name
        #     name: pihole-password
        env:
          - name: EXTERNAL_DNS_PIHOLE_PASSWORD
            valueFrom:
              secretKeyRef:
                name: pihole-password
                key: password

        args:
        - --source=service
        - --source=ingress
        # Pihole only supports A/AAAA/CNAME records so there is no mechanism to track ownership.
        # You don't need to set this flag, but if you leave it unset, you will receive warning
        # logs when ExternalDNS attempts to create TXT records.
        - --registry=noop
        # IMPORTANT: If you have records that you manage manually in Pi-hole, set
        # the policy to upsert-only so they do not get deleted.
        - --policy=upsert-only
        - --provider=pihole
        # Change this to the actual address of your Pi-hole web server
        - --pihole-server=http://pihole-web.pihole.svc.cluster.local
      securityContext:
        fsGroup: 65534 # For ExternalDNS to be able to read Kubernetes token files
```



## Deploying pihole using Yaml-Manifests
### Create a seperate namespace
```
kubectl create ns pihole
```

### Creating the Secret for the Webpassword
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pihole-webpassword
  namespace: pihole
type: Opaque
stringData:
  password: <PASSWORD_IN_PLAINTEXT>
```

```
kubectl appyl -n pihole -f <SECRET_FILE.yaml>
```

### Creating the persistentVolumeClaim
>Note: longhorn creates PersistentVolumes automatically when creating a PVC (if not otherwise specified), please refer to the documentation of your storage provider for more information.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  creationTimestamp: null
  labels:
    app: pihole
  name: pihole-config
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: longhorn-test
```

```
kubectl appyl -n pihole -f <PVC_FILE.yaml>
```

### Creating a ConfigMap for a custom dnsmasq configuration
>Note: With this configMap you create statically configured dns entries (wont't be visible inside of the pihole web interface). Useful to automate the creation process of pihole. If you don't need / want that don't create the configMap and remove the corresponding volume definition and mount in the deployment.

```yaml
apiVersion: v1
data:
  02-custom.conf: |
    address=/<FQDN_1>/<IP-ADDRESS_1>
    address=/<FQDN_2>/<IP-ADDRESS_2>
    address=/<FQDN_3>/<IP-ADDRESS_3>
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: pihole-custom-dnsmasq
```

```
kubectl appyl -n pihole -f <CONFIG-MAP_FILE.yaml>
```

### Creating the Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: pihole
  name: pihole
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: pihole
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: pihole
    spec:
      containers:
        - name: pihole
          image: pihole/pihole:<VERSION_TAG>
          imagePullPolicy: IfNotPresent
          env:
            - name: ServerIP
              value: <IP-ADDRESS>
            - name: WEBPASSWORD
              valueFrom:
                secretKeyRef:
                  name: pihole-webpassword
                  key: password
            - name: TZ
              value: 'Europe/Berlin'
          ports:
            - containerPort: 80
              name: pihole-http
              protocol: TCP
            - containerPort: 443
              name: pihole-https
              protocol: TCP
            - containerPort: 53
              name: pihole-dns-tcp
              protocol: TCP
            - containerPort: 53
              name: pihole-dns-udp
              protocol: UDP
          volumeMounts:
            - mountPath: /etc/pihole
              name: config
            - mountPath: /etc/dnsmasq.d/02-custom.conf
              name: dnsmasq-config
              subPath: 02-custom.conf
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      #schedulerName: default-scheduler
      #securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: pihole-config
        - name: dnsmasq-config
          configMap:
            name: pihole-custom-dnsmasq
            defaultMode: 420
```


```
kubectl apply -n pihole -f ./<DEPLOYMENT_FILE.yaml>
```

---

## Making pihole accessible fro outside
There are multiple ways how to do this choose **one** of the options.
>Note: I am using Metallb as a LoadBalancer. If you use another LoadBalancer you need to change the annotations accordingly.

### Using a LoadBalancer Service with a unique IP
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    metallb.universe.tf/address-pool: <NAME_OF_ADDRESS_POOL>
    metallb.universe.tf/allow-shared-ip: pihole-svc
  labels:
    app: pihole
  name: pihole-dns
  namespace: pihole
spec:
  externalTrafficPolicy: Local
  loadBalancerIP: <IP_ADDRESS>
  ports:
  - name: dns-tcp
    port: 53
    protocol: TCP
    targetPort: 53
  - name: dns-udp
    port: 53
    protocol: UDP
    targetPort: 53
  selector:
    app: pihole
  sessionAffinity: None
  type: LoadBalancer
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    metallb.universe.tf/address-pool: <NAME_OF_ADDRESS_POOL>
    metallb.universe.tf/allow-shared-ip: pihole-svc
  labels:
    app: pihole
  name: pihole-web
  namespace: pihole
spec:
  externalTrafficPolicy: Local
  loadBalancerIP: <IP_ADDRESS>
  ports:
  - name: pihole-http
    port: 80
    protocol: TCP
    targetPort: pihole-http
  - name: pihole-https
    port: 443
    protocol: TCP
    targetPort: pihole-https
  selector:
    app: pihole
  sessionAffinity: None
  type: LoadBalancer
```

```
kubectl apply -n pihole -f ./<SVC_FILE.yaml>
```

### Using an LoadBalancer with the same IP-Address as the nginx-ingress
>Note: requires configuration on the nginx-ingress controller for more information please refer to the setup guide of the nginx-ingress or the official docs. <br>
>necessary Configuration: allowing the ip of the nginx-ingress controller to be shared & setting a default certificate (optional but recommendet)

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    metallb.universe.tf/address-pool: <NAME_OF_ADDRESS_POOL>
    metallb.universe.tf/allow-shared-ip: <NGINX_INGRESS_CONTROLLER_ANNOTATION>
  labels:
    app: pihole
  name: pihole-dns
  namespace: pihole
spec:
  # externalTrafficPolicy: Local
  loadBalancerIP: <IP-Address>
  ports:
  - name: dns-tcp
    port: 53
    protocol: TCP
    targetPort: 53
  - name: dns-udp
    port: 53
    protocol: UDP
    targetPort: 53
  selector:
    app: pihole
  sessionAffinity: None
  type: LoadBalancer
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: pihole
  name: pihole-web
  namespace: pihole
spec:
  selector:
    app: pihole
  ports:
  - name: pihole-http
    port: 80
    protocol: TCP
    targetPort: pihole-http
  - name: pihole-https
    port: 443
    protocol: TCP
    targetPort: pihole-https
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pihole-ingress
  namespace: pihole
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: "<FQDN>"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: pihole-web
                port:
                  number: 80
  tls:
    - hosts:
      - <FQDN>
```
