
# Virtual machines specs

## Fedora - Workstation

    Type: e2-micro
    Location: europe-west1-d
    OS: Fedora-36

## Server - Openmediavault

    Type: e2-micro
    Location: europe-west1-b
    OS: Debian-11

## Server - Openldap

    Type: e2-micro
    Location: europe-west1-b
    OS: Fedora-36


# SSH commands

## Generate ssh key:

    ssh-keygen -b 2048 -t rsa

## Connect to any VM using:

    ssh -i private-key user@external-ip

Default location of ssh key: */home/user/.ssh*

# Installation commands

## Workstation - Fedora
------------

**IMPORTANT NOTE:**
- To turn on the virtual display device, select the **Enable display device** checkbox from the **Machine configuration > Display device** settings

------------
### Installing desktop environment 

    dnf group install "LXDE Desktop"

    systemctl set-default graphical.target 

## Creating new user for the remote desktop
------------

    useradd admin-gui

    passwd admin-gui

    usermod -a -G google-sudoers admin-gui

    usermod -a -G adm admin-gui

    usermod -a -G video admin-gui


### Installing Xrdp Server ( Remote Desktop )

    dnf -y install xrdp

    firewall-cmd --add-port=3389/tcp 

    firewall-cmd --runtime-to-permanent 

    systemctl enable --now xrdp 

**Xrdp client on linux:** Remmina (optional package required: *freerdp*)

## Server - Openmediavault
------------

    apt-get install --yes gnupg

    apt-get install --yes wget

    wget -O "/etc/apt/trusted.gpg.d/openmediavault-archive-keyring.asc" https://packages.openmediavault.org/public/archive.key

    apt-key add "/etc/apt/trusted.gpg.d/openmediavault-archive-keyring.asc"

    cat <<EOF >> /etc/apt/sources.list.d/openmediavault.list deb https://packages.openmediavault.org/public shaitan main EOF

    export LANG=C.UTF-8

    export DEBIAN_FRONTEND=noninteractive

    export APT_LISTCHANGES_FRONTEND=none

    apt-get update

    apt-get --yes --auto-remove --show-upgraded \
    --allow-downgrades --allow-change-held-packages \
    --no-install-recommends \
    --option DPkg::Options::="--force-confdef" \
    --option DPkg::Options::="--force-confold" \
    install openmediavault-keyring openmediavault

    omv-confdbadm populate

## Server - Openldap
------------

### OpenLDAP : Install

    yum -y install openldap-servers openldap-clients 

    cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG 
    
    chown ldap. /var/lib/ldap/DB_CONFIG 

    systemctl start slapd

    systemctl enable slapd

    nano /etc/hosts 
        EXTERNAL_IP server.ads.dcc server
        EXTERNAL_IP client.ads.dcc client

    slappasswd 

    vi chrootpw.ldif
        # specify the password generated above for "olcRootPW" section

        dn: olcDatabase={0}config,cn=config
        changetype: modify
        add: olcRootPW
        olcRootPW: {SSHA}LVtjdrLgXyb3PrZNOxWe1Q8zQk+zIvtz
        # pass: ads2020 

    ldapadd -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif 

    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
    
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
    
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif 

    vi chdomain.ldif 
        # replace to your own domain name for "dc=***,dc=***" section
        # specify the password generated above for "olcRootPW" section

        dn: olcDatabase={1}monitor,cn=config
        changetype: modify
        replace: olcAccess
        olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth"
        read by dn.base="cn=Manager,dc=ads,dc=dcc" read by * none

        dn: olcDatabase={2}hdb,cn=config
        changetype: modify
        replace: olcSuffix
        olcSuffix: dc=ads,dc=dcc

        dn: olcDatabase={2}hdb,cn=config
        changetype: modify
        replace: olcRootDN
        olcRootDN: cn=Manager,dc=ads,dc=dcc

        dn: olcDatabase={2}hdb,cn=config
        changetype: modify
        add: olcRootPW
        olcRootPW: {SSHA}LVtjdrLgXyb3PrZNOxWe1Q8zQk+zIvtz

        dn: olcDatabase={2}hdb,cn=config
        changetype: modify
        add: olcAccess
        olcAccess: {0}to attrs=userPassword,shadowLastChange by
        dn="cn=Manager,dc=ads,dc=dcc" write by anonymous auth by self write by * none
        olcAccess: {1}to dn.base="" by * read
        olcAccess: {2}to * by dn="cn=Manager,dc=ads,dc=dcc" write by * read

    ldapmodify -Y EXTERNAL -H ldapi:/// -f chdomain.ldif 

    vi basedomain.ldif
        # replace to your own domain name for "dc=***,dc=***" section

        dn: dc=ads,dc=dcc  
        objectClass: top
        objectClass: dcObject
        objectclass: organization
        o: Server World
        dc: ads

        dn: cn=Manager,dc=ads,dc=dcc  
        objectClass: organizationalRole
        cn: Manager
        description: Directory Manager

        dn: ou=People,dc=ads,dc=dcc
        objectClass: organizationalUnit
        ou: People

        dn: ou=Group,dc=ads,dc=dcc  
        objectClass: organizationalUnit
        ou: Group

    ldapadd -x -D cn=Manager,dc=ads,dc=dcc -W -f basedomain.ldif 

    firewall-cmd --add-service=ldap --permanent 

    firewall-cmd --reload 

External link: https://www.server-world.info/en/note?os=CentOS_7&p=openldap&f=1

### phpLDAPadmin : Install

    yum --enablerepo=epel -y install phpldapadmin 

    vi /etc/phpldapadmin/config.php
        # line 397: uncomment, line 398: comment out

        $servers->setValue('login','attr','dn');
        //
        $servers->setValue('login','attr','uid'); 

    vi /etc/httpd/conf.d/phpldapadmin.conf 
        # line 12: add access permission
        Require all granted

    systemctl restart httpd 
    
External link: https://www.server-world.info/en/note?os=CentOS_7&p=openldap&f=7

# Dasboard

**Openmediavault :** (only available after [Installation commands / Server - Openmediavault ](#installation-commands))

    http://external-ip-of-openmediavault-vm/#/dashboard

Credentials:

    user: admin
    password: openmediavault

**Openldap :** (only available after [Installation commands / Server - Openldap ](#installation-commands))

    http://external-ip-of-openldap/ldapadmin/

Credentials:

    user: cn=Manager,dc=ads,dc=dcc
    password: ads2020
