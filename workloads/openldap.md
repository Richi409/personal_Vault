# OpenLDAP

## Installation
- Install the necessary packages
    ```bash
    sudo apt install slapd ldap-utils
    ```
- You do not need to provide a password during installation

## Configuration
### Replacing rfc2307bis schema
> making the `posixGroup` objectClass non-structural
- download the schema
    ```bash
    curl https://raw.githubusercontent.com/mesosphere-backup/docker-containers/refs/heads/master/openldap/rfc2307bis/rfc2307bis.ldif | sudo tee /etc/ldap/schema/rfc2307bis.ldif
    ```
- edit the file `/usr/share/slapd/slapd.init.ldif`
  - comment the line `include: file:///etc/ldap/schema/nis.ldif`
  - add the line `include: file:///etc/ldap/schema/rfc2307bis.ldif`
- run `sudo dpkg-reconfigure -plow slapd`
    - Omit OpenLDAP Server Configuration? $\Rightarrow$ **NO**
    - DNS domain Name? $\Rightarrow$ **\<your-domain\>.\<root-domain\>**
    - Admin Password? $\Rightarrow$ provide a admin password
        - do not use any special characters if you plan to use `phpldapadmin`
    - Remove Database when purging slapd? $\Rightarrow$ **YES**
    - Move Old Database? $\Rightarrow$ **YES**
- verify the used schemas
    ```bash
    sudo ldapsearch -LLL -Y external -H ldapi:/// -b "cn=schema,cn=config" -s one dn
    ```
    ```bash
    ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "cn=config" dn
    ```

### Configure LDAPs using TLS
1. Create a certificate following the certbot guide
2. Add Certificates using ldif
    ```ldif
    dn: cn=config
    changetype: modify
    replace: olcTLSCipherSuite
    olcTLSCipherSuite: NORMAL
    -
    replace: olcTLSCRLCheck
    olcTLSCRLCheck: none
    -
    replace: olcTLSVerifyClient
    olcTLSVerifyClient: never
    -
    replace: olcTLSCertificateKeyFile
    olcTLSCertificateKeyFile: </path/to/cert_key/file>
    -
    replace: olcTLSCertificateFile
    olcTLSCertificateFile: </path/to/cert/file>
    -
    replace: olcTLSProtocolMin
    olcTLSProtocolMin: 3.3
    ```
3. Edit the file `/etc/default/slapd`
    - Make sure `ldaps://` is configured (Optional: disable `ldap://`
        ```
        SLAPD_SERVICES="ldaps:/// ldapi:///"
        ```
### Loading additional schemas
- toDo

### adding modules
- toDo

### Adding Rules to the ACL
- toDo

### Basic LDAP structure
- toDo

## Webinterface phpldapadmin
### Installation
```bash
sudo apt install phpldapadmin
```
### Configuration
- Configuration File: `/etc/phpldapadmin`
- edit the following values:
    - Servername
        ```php
        $servers->setValue('server','name','<Servername>');
        ```
    - Connection to LDAP Server
        - use the socket if running on the same system as the LDAP Server
            ```php
            $servers->setValue('server','host','ldapi://%2frun%2fslapd%2fldapi');
            ```
    - set BaseDN
        ```php
        $servers->setValue('server','base',array('dc=<domain>,dc=<domain>'));
        ```
    - DN of Bind-User (Admin)
        ```php
        $servers->setValue('login','bind_id','cn=admin,dc=<domain>,dc=<domain>');
        ```
