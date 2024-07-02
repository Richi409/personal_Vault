# How to setup nginx-ingress
> You need to have [Helm](./helm_setup.md) installed.

For Reference also see the [Guide](https://kubernetes.github.io/ingress-nginx/deploy/) from Kubernetes and check out the official GitHub [repository](https://github.com/kubernetes/ingress-nginx).

## Adding helm repository
- You can add the needed repo with the following command:
    ```
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    ```
- You can now search for the available versions with ths command:
    ```
    helm search repo ingress-nginx --versions
    ```
    
## Installing with Helm

1. Create a custom `values.yaml`
- setting the default certificate
- setting static loadbalancer ip
- specifying address-pool used by the loadbalancer service
- marking the loadbalancer to allow sharing the ip with other loadbalancer services

    ```yaml
    controller:
      extraArgs:
        default-ssl-certificate: "default/acmedns-certificate"
      service:
        loadBalancerIP: "192.168.2.20"
        annotations:
          metallb.universe.tf/address-pool: "addr-pool"
          metallb.universe.tf/allow-shared-ip: "ingress-nginx-loadbalancer"
    ```
2. Install nginx-ingress
    ```
    helm install ingress-nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx \
    --version <CHART_VERSION> \
    --namespace ingress-nginx --create-namespace\
    -f ./values.yaml
    ```

### Upgrading deployment
```yaml
helm upgrade ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --version <CHART_VERSION> --namespace ingress-nginx -f ./values.yaml
```

### Updating (upgrading) Configuration
```yaml
helm upgrade -n ingress-nginx ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx -f ./values.yaml
```


## Alternatively create a template with helm
- Create the template as follows:
    ```
    helm template ingress-nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx \
    --version <CHART_VERSION> \
    --namespace ingress-nginx \
    > ./nginx-ingress_<APP_VERSION>.yaml
    ```

### Create the namespace
```
kubectl create namespace ingress-nginx
```

### Apply the manifest file
> if you wish to assign a static IP to your nginx-ingress-controller then add `loadBalancerIP: <IP-Address>` to your LoadBalancer Service under `spec:` <br/>
> generally edit the manifest for custom settings
```
kubectl aplly -f /path/to/nginx-ingress_<APP_VERSION>.yaml
```

## Example Service Using the ingress controller
Make sure to set `ingressClassName to nginx` this tells kubernetes to use the nginx-ingress controller as the ingress controller

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deploy
spec:
  replicas: 1
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: example-app
  template :
    metadata:
      labels:
        app: example-app
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: example-svc
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: example-app
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-example-svc
  # annotations:
  #   nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: "test.homelab.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example-svc
                port:
                  number: 80
```
