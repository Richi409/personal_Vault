# How to setup cert-manager
Please refer to the [official Documentation](https://cert-manager.io/docs/installation/) for additional information.

## Installing cert-manager using Helm
1. Add the repo
    ```
    helm repo add jetstack https://charts.jetstack.io --force-update
    ```
2. Searchin for the newest version
    ```
    helm search repo jetstack/cert-manager --versions | head -2
    ```
3. Using the helm chart to install cert-manager
    ```
    helm install \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version <VERSION> \
    --set crds.enabled=true
    ```

## Issuing Certificates
> Refer to this [Article](https://cert-manager.io/docs/usage/ingress/) for useful information
1. Register at an ACMEDNS Server
    ```
    curl -X POST https://auth.acme-dns.io/register
    ```
2. you should get back a json string similar to this
    ```json
    {
    "username": "<id_1>",
    "password": "<id_2>",
    "fulldomain": "<id_3>.auth.acme-dns.io",
    "subdomain": "<id_3>",
    "allowfrom": []
    }
    ```
    - give it a key of your desired domain like this and save it to the file `acmedns.json'
        ```json
        { "<fqdn>":
            {
            "username": "<id_1>",
            "password": "<id_2>",
            "fulldomain": "<id_3>.auth.acme-dns.io",
            "subdomain": "<id_3>",
            "allowfrom": []
            },
        }
        ```
3. Set CNAME
    - create the CNAME `_acme-challenge` with the value of the `fulldomain` field
4. Create a secret
    - Use the json file from Step 2 for this
    ```
    kubectl create secret generic acme-dns --from-file acmedns.json
    ```
5. Create an Issuer and a Certificate
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
        name: acme-issuer
    spec:
        acme:
            email: <email>
            # server: https://acme-v02.api.letsencrypt.org/directory
            server: https://acme-staging-v02.api.letsencrypt.org/directory
            privateKeySecretRef:
                name: cm-le-key
            solvers:
            - dns01:
                acmeDNS:
                    host: https://auth.acme-dns.io
                    accountSecretRef:
                        name: acme-dns
                        key: acmedns.json
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
        name: acmedns-certificate
    spec:
        issuerRef:
            name: acme-issuer
        secretName: acmedns-certificate
        dnsNames:
            - "<domain 1>"
            - "<domain 2>"
    ```

---

## Issuing a self-signed Certificate
1. Creating a Self Signed Issuer
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
        name: self-signed-issuer
    spec:
        selfSigned: {}
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
        name: self-signed-cert
        namespace: cert-manager
    spec:
        isCA: true
        duration: "43800h" # 5 Years
        commonName: rm-hlab.com
        secretName: rm-hlab-com-key-pair
        privateKey:
            algorithm: ECDSA
            size: 256
        issuerRef:
            name: self-signed-issuer
            kind: ClusterIssuer
            group: cert-manager.io
    ```
2. Create a Certificate signed by the CA
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
        name: ss-ca-issuer
    spec:
        ca:
            secretName: rm-hlab-com-key-pair
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
        name: staging
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
        name: test1-rm-hlab-com
        namespace: staging
    spec:
        isCA: false
        duration: "2160h" # 90d
        renewBefore: "360h" # 15d
        commonName: test1.rm-hlab.com
        dnsNames:
            - test1.rm-hlab.com
            - www.test1.rm-hlab.com
        secretName: test1-rm-hlab-com-key-pair
        privateKey:
            algorithm: RSA
            encoding: PKCS1
            size: 4096
        issuerRef:
            name: ss-ca-issuer
            kind: ClusterIssuer
            group: cert-manager.io
```
