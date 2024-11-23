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
- run `sudo dpkg-reconfigure slapd`
    - Omit OpenLDAP Server Configuration? $\Rightarrow$ **NO**
    - DNS domain Name? $\Rightarrow$ **\<your-domain\>.\<root-domain\>**
    - Admin Password? $\Rightarrow$ provide a admin password
    - Remove Database when purging slapd? $\Rightarrow$ **YES**
    - Move Old Database? $\Rightarrow$ **YES**
- verify the used schemas
    ```bash
    sudo ldapsearch -LLL -Y external -H ldapi:/// -b cn=schema,cn=config -s one dn
    ```

