# How to setup MetalLB as LoadBalancer

You can find a helpful YouTube Video [here](https://www.youtube.com/watch?v=2hVAHYUrOgQ)

> Check on [github](https://github.com/metallb/metallb) for new versions
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

## Create a IP-Address Pool
- The IP-Addresses under `addresses` can be ranges, IP-Addresses with CIDR's or single IP's.
- These IP-Addresses are IP's from your local network (imagine it like a bridge network in linux that can only assign specific ip's to its "VM's"
- Therefore these IP-Addresses should not be in the DHCP-Range (and you need to be careful not to assign these IP's manually e.g. static IP for a server)

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: addr-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.2.20-192.168.2.30
    # - 2001:470:61bb:10::210-2001:470:61bb:10::220
  autoAssign: true
```

---

## Create a Layer 2 Advertisement
- By default your local network will not know when MetalLB will assign a "Public"-IP, for that to happen you need to explicitly advertise these IP-Addresses to the rest of the network. (There are different Options to L2Advertisement)

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2adv
  namespace: metallb-system
```

---

## Example Deployment with LoadBalancer Service
### Deployment
```yaml
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - image: nginx:1.27.0
          name: nginx
          ports:
            - containerPort: 80
```

### LoadBalancer Service
> Use `ipFamilyPolicy: PreferDualStack` to enable IPV6

- The Service should get a Public IP-Address (in the range defined above in the IPAddressPool)
- You should now be able to access the the nginx container on the assigned IP-Address and the Port 80
    - you can look for the assigned IP-Address with the command `kubectl get svc -o wide`
- To request a specific IP-Address use the LoadBalancerIPs Annotation commented out below.
- To select a specific address-pool from multiple pools use the address-pool Annotation (with the apropriate address-pool name) commented out below

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
  # annotations:
  #   metallb.universe.tf/LoadBalancerIPs: 192.168.2.25, 2001:470:61bb:10::220
  #   metallb.universe.tf/address-pool: addr-pool
spec:
  # ipFamilyPolicy: PreferDualStack
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - name: '80'
      protocol: TCP
      port: 80
      targetPort: 80
```
