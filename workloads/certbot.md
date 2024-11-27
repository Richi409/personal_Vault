# Certbot Setup

More information can be found on Github: [[ joohoi/acme-dns ]](https://github.com/joohoi/acme-dns-certbot-joohoi)

## Installation and Setup
1. Install Certbot
    ```bash
    sudo apt install certbot
    ```
2. Install `python-requests` library
    ```bash
    sudo apt install python3-requests
    ```
3. Download the authentication-hook
    ```bash
    sudo curl -o /etc/letsencrypt/acme-dns-auth.py https://raw.githubusercontent.com/joohoi/acme-dns-certbot-joohoi/master/acme-dns-auth.py
    ```
    ```bash
    sudo chmod 0700 /etc/letsencrypt/acme-dns-auth.py
    ```
4. Change the interpreter
    ```bash
    #!/usr/bin/env/ python3
    ```
5. Change parameters in `/etc/letsencrypt/acme-dns-auth.py` if needed
    ```
    # URL to acme-dns instance
    ACMEDNS_URL = "https://auth.acme-dns.io"

    # Path for acme-dns credential storage
    STORAGE_PATH = "/etc/letsencrypt/acmedns.json"

    # Whitelist for address ranges to allow the updates from
    # Example: ALLOW_FROM = ["192.168.10.0/24", "::1/128"]
    ALLOW_FROM = []

    # Force re-registration. Overwrites the already existing acme-dns accounts.
    FORCE_REGISTER = False
    ```
6. Issue Certificate
    - run command
        ```bash
        sudo certbot certonly --manual --manual-auth-hook /etc/letsencrypt/acme-dns-auth.py --preferred-challenges dns --debug-challenges -d <domain> -d <second_domain>
        ```
        ```bash
        sudo certbot certonly --manual --manual-auth-hook /etc/letsencrypt/acme-dns-auth.py --preferred-challenges dns --debug-challenges -d test.com -d \*.test.com
        ```
    - provide e-mail address
    - accept terms of service $\Rightarrow$ `Y`
    - do not share e-mail address $\Rightarrow$ `N`
    - create CNAME DNS Record when promted
        - `_acme-challenge.<base>.<domain>` to `<generated CNAME value printed on console>`
        - wait for 2-5 min for DNS propagation
        - press Enter to continue

## Additional Configuration
### making the certificate accessible
```bash
sudo chmod 755 /etc/letsencrypt/live/
```

### perforing actions after certificate renewal
- edit the file `/etc/letsencrypt/renewal/<your.domain>.conf`
- add `renew_hook = <command or script_path>` under `[renewalparams]:`
    ```bash
    renew_hook = cp /etc/letsencrypt/live/<domain>/* /etc/ldap/ssl/ && chown openldap:openldap /etc/ldap/ssl/* && systemctl restart slapd
    ```
    ```bash
    renew_hook = systemctl reload nginx
    ```
