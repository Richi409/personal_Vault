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
#### dyngroup schema
```ldif
# dyngroup.schema -- Dynamic Group schema
# $OpenLDAP$
## This work is part of OpenLDAP Software <http://www.openldap.org/>.
##
## Copyright 1998-2024 The OpenLDAP Foundation.
## All rights reserved.
##
## Redistribution and use in source and binary forms, with or without
## modification, are permitted only as authorized by the OpenLDAP
## Public License.
##
## A copy of this license is available in the file LICENSE in the
## top-level directory of the distribution or, alternatively, at
## <http://www.OpenLDAP.org/license.html>.
#
# Dynamic Group schema (experimental), as defined by Netscape.  See
# http://www.redhat.com/docs/manuals/ent-server/pdf/esadmin611.pdf
# page 70 for details on how these groups were used.
#
# A description of the objectclass definition is available here:
# http://www.redhat.com/docs/manuals/dir-server/schema/7.1/oc_dir.html#1303745
#
# depends upon:
#       core.schema
#
# These definitions are considered experimental due to the lack of
# a formal specification (e.g., RFC).
#
# NOT RECOMMENDED FOR PRODUCTION USE!  USE WITH CAUTION!
#
# The Netscape documentation describes this as an auxiliary objectclass
# but their implementations have always defined it as a structural class.
# The sloppiness here is because Netscape-derived servers don't actually
# implement the X.500 data model, and they don't honor the distinction
# between structural and auxiliary classes. This fact is noted here:
# http://forum.java.sun.com/thread.jspa?threadID=5016864&messageID=9034636
#
# In accordance with other existing implementations, we define it as a
# structural class.
#
# Our definition of memberURL also does not match theirs but again
# their published definition and what works in practice do not agree.
# In other words, the Netscape definitions are broken and interoperability
# is not guaranteed.
#
# Also see the new DynGroup proposed spec at
# http://tools.ietf.org/html/draft-haripriya-dynamicgroup-02
dn: cn=dyngroup,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: dyngroup
olcObjectIdentifier: {0}NetscapeRoot 2.16.840.1.113730
olcObjectIdentifier: {1}NetscapeLDAP NetscapeRoot:3
olcObjectIdentifier: {2}NetscapeLDAPattributeType NetscapeLDAP:1
olcObjectIdentifier: {3}NetscapeLDAPobjectClass NetscapeLDAP:2
olcObjectIdentifier: {4}OpenLDAPExp11 1.3.6.1.4.1.4203.666.11
olcObjectIdentifier: {5}DynGroupBase OpenLDAPExp11:8
olcObjectIdentifier: {6}DynGroupAttr DynGroupBase:1
olcObjectIdentifier: {7}DynGroupOC DynGroupBase:2
olcAttributeTypes: {0}( NetscapeLDAPattributeType:198 NAME 'memberURL' DESC 'I
 dentifies an URL associated with each member of a group. Any type of labeled
 URL can be used.' SUP labeledURI )
olcAttributeTypes: {1}( DynGroupAttr:1 NAME 'dgIdentity' DESC 'Identity to use
  when processing the memberURL' SUP distinguishedName SINGLE-VALUE )
olcAttributeTypes: {2}( DynGroupAttr:2 NAME 'dgAuthz' DESC 'Optional authoriza
 tion rules that determine who is allowed to assume the dgIdentity' EQUALITY a
 uthzMatch SYNTAX 1.3.6.1.4.1.4203.666.2.7 X-ORDERED 'VALUES' )
olcAttributeTypes: {3}( DynGroupAttr:3 NAME 'dgMemberOf' DESC 'Group that the
 entry belongs to' EQUALITY distinguishedNameMatch SYNTAX 1.3.6.1.4.1.1466.115
 .121.1.12 )
olcObjectClasses: {0}( NetscapeLDAPobjectClass:33 NAME 'groupOfURLs' SUP top S
 TRUCTURAL MUST cn MAY ( memberURL $ businessCategory $ description $ o $ ou $
  owner $ seeAlso $ member ) )
olcObjectClasses: {1}( DynGroupOC:1 NAME 'dgIdentityAux' SUP top AUXILIARY MAY
  ( dgIdentity $ dgAuthz ) )
```

#### openssh-lpk schema
```ldif
# AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
# CRC32 f6bf57a2
dn: cn=openssh-lpk,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: openssh-lpk
olcAttributeTypes: {0}( 1.3.6.1.4.1.24552.500.1.1.1.13 NAME 'sshPublicKey' DES
 C 'MANDATORY: OpenSSH Public key' EQUALITY octetStringMatch SYNTAX 1.3.6.1.4.
 1.1466.115.121.1.40 )
olcObjectClasses: {0}( 1.3.6.1.4.1.24552.500.1.1.2.0 NAME 'ldapPublicKey' DESC
  'MANDATORY: OpenSSH LPK objectclass' SUP top AUXILIARY MAY ( sshPublicKey $
 uid ) )
 ```

### adding modules
#### dynlist module
```ldif
dn: cn=module,cn=config
cn: module
objectClass: olcModuleList
objectClass: top
olcmoduleload: dynlist.la
olcmodulepath: /usr/lib/ldap

dn: olcOverlay=dynlist,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcDynamicList
olcOverlay: dynlist
olcDlAttrSet: groupOfURLs memberURL member+memberOf@groupOfNames
```

### Adding Rules to the ACL
```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to dn.exact="dc=<domain>,dc=<domain>" by * read
olcAccess: {1}to attrs=userPassword by self write by dn="cn=password-self-service,ou=applications,dc=<domain>,dc=<domain>" write by dn.children="ou=applications,dc=rm-hab,dc=com" read by anonymous auth by * none
olcAccess: {2}to attrs=sshPublicKey by self write by dn="cn=password-self-service,ou=applications,dc=<domain>,dc=<domain>" write by dn.children="ou=applications,dc=rm-hab,dc=com" read by anonymous auth by * none
olcAccess: {3}to attrs=shadowLastChange by self write by * read
olcAccess: {4}to dn.base="dc=<domain>,dc=<domain>" by dn.children="ou=applicatons,dc=<domain>,dc=<domain>" read
olcAccess: {5}to dn.base="ou=groups,dc=<domain>,dc=<domain>" by dn.children="ou=applications,dc=<domain>,dc=<domain>" read
olcAccess: {6}to dn.children="ou=groups,dc=<domain>,dc=<domain>" by dnattr=owner read
olcAccess: {7}to dn.subtree="ou=users,dc=<domain>,dc=<domain>" by dn.children="ou=applications,dc=<domain>,dc=<domain>" read
```

### Basic LDAP structure
```ldif
dn: ou=users,dc=<domain>,dc=<domain>
ou: users
objectClass: top
objectClass: organizationalUnit

dn: ou=groups,dc=<domain>,dc=<domain>
ou: groups
objectClass: top
objectClass: organizationalUnit

dn: ou=applications,dc=<domain>,dc=<domain>
ou: applications
objectClass: top
objectClass: organizationalUnit

dn: ou=policies,dc=<domain>,dc=<domain>
ou: users
objectClass: top
objectClass: organizationalUnit
```

### Adding Users
- generate Password hash
    ```bash
    sudo slappasswd -h '{SSHA}'
    ```

#### normal User
```ldif
dn: cn=<first name + last name>,ou=users,dc=<domain>,dc=<domain>
objectClass: inetOrgPerson
objectClass: ldapPublicKey
objectClass: organizationalPerson
objectClass: person
objectClass: posixAccount
cn: <first name + last name>
givenName: <first name>
sn: <last name>
userPassword: <hashed password>
mail: <mail>
uid: <username>
uidNumber: <uidNumber>
gidNumber: <gidNumber>
loginShell: /bin/bash
homeDirectory: /home/<username>
```

#### bind User
```ldif
dn: cn=<username>,ou=applications,dc=<domain>,dc=<domain>
cn: <username>
objectClass: simpleSecurityObject
objectClass: organizationalRole
userPassword: <hashed password>
```

### Creating a Group
```ldif
dn: cn=<group_name>,ou=groups,dc=<domain>,dc=<domain>
objectClass: posixGroup
objectClass: top
objectClass: groupOfURLs
cn: <group_name>
gidNumber: <gidNumber>
memberURL: ldap:///<base>??<scope>?<filter>
description: <description>
owner: <dn of owner Object>
```

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

### Changing Webserver to nginx
- disable `apache2` webserver
    ```bash
    sudo systemctl disable apache2
    ```
- stop `apache2` webserver
    ```bash
    sudo systemctl stop apache2
    ```
- install nginx
    ```bash
    sudo apt install nginx
    ```
- enable and start nginx
    ```bash
    sudo systemctl enable --now nginx
    ```
- install needed php module
    ```bash
    sudo apt install php8.2-fpm
    ```
- add a `redirect_to_https.conf` to `/etc/nginx/sites-available`
    ```
    server {
    	listen 80 default_server;
    
    	server_name _;
    
    	return 301 https://$host$request_uri;
    }
    ```
- add a `ldap.<domain>.<domain>.conf` to `/etc/nginx/sites-available`
    ```
    server {
    	listen 443 ssl;
    	server_name <fqdn>;
    
    	ssl_certificate </path/to/certificate>;
    	ssl_certificate_key </path/to/certificate_key>;
    
    	location / {
    		index index.php index.html index.htm;
    		root /usr/share/phpldapadmin/htdocs;
    		location ~ ^/(.+\.php)$ {
    			try_files $uri = 404;
    			if ($request_filename !~* htdocs) {
    				rewrite ^/(/.*)?$ /htdocs$1;
    			}
    			fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
    			fastcgi_index index.php;
    			fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    			include /etc/nginx/fastcgi_params;
    		}
    	}
    }
    ```
- enable both configs by creating symlinks
    - redirect_to_https:
    ```bash
    ln -s /etc/nginx/sites-available/redirect_to_https.conf /etc/nginx/sites-enabled/redirect_to_https.conf
    ```
    - ldap.\<domain\>.\<domain\>.conf:
    ```bash
    ln -s /etc/nginx/sites-available/ldap.<domain>.<domain>.conf /etc/nginx/sites-enabled/ldap.<domain>.<domain>.conf
    ```
- test configuration
    ```bash
    sudo nginx -t
    ```
- apply configuration (when tests complete without errors)
    ```bash
    sudo nginx -s reload
    ```
