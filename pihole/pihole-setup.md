# How to setup pihole in Kubernetes
You can install pihole using the [pihole-kubernetes](https://github.com/MoJo2600/pihole-kubernetes) Repository...
```
helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
helm repo update
```
or create the manifest files by yourself.

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
