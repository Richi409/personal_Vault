# How to setup cert-manager
Please refer to the [official Documentation](https://cert-manager.io/docs/installation/) for additional information.

## Installing cert-manager using Helm
1. Add the repo
    ```
    helm repo add jetstack https://charts.jetstack.io --force-update
    ```
2. Using the helm chart to install cert-manager
    ```
    helm install \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version <VERSION> \
    --set crds.enabled=true
    ```

## Issuing Certificates
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
