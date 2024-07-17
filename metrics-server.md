# How to install metrics-server

1. Add the Helm Repo
    ```
    helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
    ```
2. Create a `values.yaml`
    ```yaml
    args:
      - --kubelet-insecure-tls
    ```
3. Install the metrics-server
    ```
    helm upgrade --install metrics-server metrics-server/metrics-server -f values.yaml --namespace metrics-server --create-namespace
    ```
