# Setup of Nginx LoadBalancer
If you have multiple master nodes you need to setup a LoadBalancer between them and use this LoadBalancer as api-enpoint while creating the cluster.

1. Install nginx and the stream-module
    ```
    sudo apt install nginx libnginx-mod-stream
    ```
2. enable nginx
    ```
    systemctl enable nginx
    ```
3. edit the `nginx.conf`
    - located under `/etc/nginx/nginx.conf`
    - add this line at the bottom
        ```
        include /etc/nginx/streams.d/*.conf;
        ```
4. create the folder `/etc/nginx/streams.d`
   ```
   mkdir /etc/nginx/streams.d
   ```
5. create this configuration file `/etc/nginx/streams.d/k8s-lb.conf`
   ```
    stream {
    	upstream <UPSTREAM_NAME> {
    		server <MASTER_NODE_1>:6443;
    		server <MASTER_NODE_2>:6443;
    		server <MASTER_NODE_3>:6443;
    	}
    
    	server {
    		listen 6443;
    
    		proxy_pass <UPSTREAM_NAME>;
    	}
    }
   ```
   > add additional master nodes to the upstream accordingly to your cluster
6. Test configuration
    ```
    nginx -t
    ```
7. Reload nginx
    ```
    nginx -s reload
    ```
