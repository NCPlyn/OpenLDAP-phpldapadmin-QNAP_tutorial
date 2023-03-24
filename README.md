## OpenLDAP server:
- Installation\
    `sudo apt update`\
    `sudo apt install slapd ldap-utils`\
    `sudo systemctl enable slapd`
- Configure\
    `sudo dpkg-reconfigure slapd`
    - Omit configuration: **NO**
    - DNS Domain: (domain of the server) ex.: **slapdserver.organiz.com**
    - Organization name: ex.: **organiz**
    - Administrator password: ...
    - Purge: **NO**
    - Move old database: **YES**

    `sudo systemctl restart slapd`
    
- If you want to use SSH keys:
    - Create schema file
     ```
	cat << EOL >~/openssh-lpk.ldif
	dn: cn=openssh-lpk,cn=schema,cn=config
	objectClass: olcSchemaConfig
	cn: openssh-lpk
	olcAttributeTypes: ( 1.3.6.1.4.1.24552.500.1.1.1.13 NAME 'sshPublicKey'
	  DESC 'MANDATORY: OpenSSH Public key'
	  EQUALITY octetStringMatch
	  SYNTAX 1.3.6.1.4.1.1466.115.121.1.40 )
	olcObjectClasses: ( 1.3.6.1.4.1.24552.500.1.1.2.0 NAME 'ldapPublicKey' SUP top AUXILIARY
	  DESC 'MANDATORY: OpenSSH LPK objectclass'
	  MAY ( sshPublicKey $ uid )
	  )
	EOL
    ```
    - Add schema to OpenLDAP config\
    `ldapadd -Y EXTERNAL -H ldapi:/// -f ~/openssh-lpk.ldif`
- Test your LDAP configuration
    - (Replace with valid DCs of your domain)\
    ```ldapsearch -x -W -D cn=admin,dc=slapdserver,dc=organiz,dc=com -b dc=slapdserver,dc=organiz,dc=com -LLL```
    - If your configuration is working, you will be returned with listed configuration of your LDAP server, otherwise you get an error.
### phpLDAPadmin
- Installation\
    `sudo apt install phpldapadmin`
- Configure\
    `sudo nano /etc/phpldapadmin/config.php`
    - Find `$servers->setValue('server','base'`
    and replace the third parameter with your DC (Domain controller) ex.:
    `$servers->setValue('server','base',array('dc=slapdserver,dc=organiz,dc=com'));`
    - Find `$servers->setValue('login','bind_id'`
    and replace `cn=Manager` with `cn=admin` in the third parameter and fill out your DC. ex.:
    `$servers->setValue('login','bind_id','cn=admin,dc=slapdserver,dc=organiz,dc=com');`

    (You don't have to do this step but it will be easier to make user accounts)
    Copy & replace the posixAccount template with edited one in this repository.
    - `sudo cp posixAccount.xml /etc/phpldapadmin/templates/creation/posixAccount.xml`
    
    `sudo systemctl restart apache2`

## LDAP Configuration using phpLDAPadmin
- Visit your ldap server web, ex.:
  `http://slapdserver.organiz.com/phpldapadmin`
- The login form should be automaticaly filled, just enter you Administrator password
- Create two new **Organisational Units** named "users" and "groups"
- Create a new **Posix Group** item under "groups" OU named "users"
- Create a new **User Account** item under "users" OU, more details more down in "**How to add new user**"

## Client machine configuration for LDAP
- Installation\
    `sudo apt install ldap-auth-client nscd nfs-common`
    - Enter IP or domain of your LDAP server, ex.: `ldap://slapdserver.organiz.com`
    - Enter DC of your LDAP server, ex.: `dc=slapdserver,dc=organiz,dc=com`
    - LDAP version: **3**
    - Local database admin?: **YES**
    - Local database login?: **NO**
    - LDAP Account for root, ex.: `cn=admin,dc=slapdserver,dc=organiz,dc=com`
    - LDAP root password: *Administrator password you entered into the slapd configuration*
- Configuration
    - `sudo nano /etc/nsswitch.conf`
        - Add "ldap" to passwd,group & shadow at every end of the line
    - `sudo nano /etc/pam.d/common-session`
        - Add this to the end of the file (makes homedir if it doesn't exists):\
        `session required    pam_mkhomedir.so skel=/etc/skel umask=0022`
    - `sudo pam-auth-update`
        - Check every item and Apply
    - `sudo systemctl restart nscd`
    
    If you configured everything right, you should see LDAP accounts in the return of `getent passwd` command
    
    - To make passwd work with ldap accounts, remove `use_authtok` in `/etc/pam.d/common-password`

    If you are going to use NFS remote storage for homes (make sure the nfs&ldap on NAS is done first before doing this):
    - `sudo apt install nfs-common`
    - Create folder for the homes you specified while creating the User Account, ex.: `sudo mkdir /homenas`
    - `sudo nano /etc/fstab`
        - Add this to the end of your file with the right domain/IP of the server and folder, ex.: `nas.organiz.com:/servery /homenas nfs defaults 0 0`

- If you want to use SSH keys:
    ```
    sudo apt install python3 python3-pip python3-ldap
    sudo pip3 install ssh-ldap-pubkey
    sh -c 'echo "AuthorizedKeysCommand /usr/local/bin/ssh-ldap-pubkey-wrapper\nAuthorizedKeysCommandUser nobody" >> /etc/ssh/sshd_config' && service ssh restart
    ```
    
- Reboot client machine

## How to add new user
- In phpLDAPadmin, create new child entry 'User account' under ou 'users'
- Fill out the form, select /bin/bash as Login shell, (If you want to have the user homes on remote NFS storage, make their home is for example `/homenas/their-usrname`) (also, it will get autofilled once you enter some names). Make sure their UID doesn't match any existing UID on the client machines' and Create Object.
- If using SSHKey:
    - In the newly created user, go to 'objectClass' attribute section, click 'add value', choose the “ldapPublicKey” attribute and Update Object.
    - Now in the user edit page, click 'Add new attribute' on the top part, choose 'sshPublicKey', paste their public key into the text area and Update Object.
- Login into the account on any machine (it will create it's homedir). (User can change their password using 'passwd')
- If using QNAP NAS as homedir:
    - Login into the QNAP and open the permissions tab for their folder in /servery
    - **Remove any group or user that isn't the user who created the folder or administrator**
    - Do this for every existing server:
       - With sudo account, move their data from /home folder on the server to the NAS `sudo cp -p -r /home/xxxx/* /homenas/xxxx/`
       - Set the right permissions for their files with their LDAP uid `sudo chown -R ldapuserid:500 /homenas/xxxx/`
- Optional: Copy the users crypted password from /etc/shadow (between the first two ":") and add "{CRYPT}" before it
    - In phpldapadmin, in the users password parameter, select encrypting algorithm "crypt" and paste the encrypted password with "{CRYPT}" infront of it and "Update Object"
- On every server, check if the LDAP user exists with `getent passwd`, then delete the converted user entry in `/etc/passwd` & `/etc/shadow` & `/etc/group` & their /home/xxxx folder, but you can keep it in case of backup

## QNAP NAS Configuration

### LDAP:
- control panel -> oprávnění -> zabezpečení domén
	- zvolit autentizace ldap
	- typ serveru: vzdálený server ldap
	- hostitelský server: *doména nebo ip ldap serveru*
	- zabezpečení ldap: ldap://
	- základní DN: *DC ldap serveru ex.: `dc=slapdserver,dc=organiz,dc=com`*
	- Root DN: *DC ldap serveru a cn admina ec.: `cn=admin,dc=slapdserver,dc=organiz,dc=com`*
	- Root Heslo: ....
	- Základní uživatelský DN: *DC ldap serveru a ou users, ex.: `ou=users,dc=slapdserver,dc=organiz,dc=com`*
	- Základní skupinový DN: *DC ldap serveru a ou groups, ex.: `ou=groups,dc=slapdserver,dc=organiz,dc=com`*
- Použít

- měli by se objevit uživetelé v Opravávnění->Uživatelé, když přepnete Místní uživatelé na Uživatelé domény

### NFS:
- vytvořte složku pro soubory - /servery
- control panel -> sdílené složky
	- v "Pokročilá práva" zaškrtněte "Povolit práva pokročilých složek"
	- Poté zpátky klitněte na "upravit oprávnění sdílené složky" u složky "servery"
	- Přidejte zde doménovou skupinu "users" s právy R/W, odškrtněte "aplikovat na soubory a podsložky"

- **tohle vždy při přidání nového uživatele zkontrolovat:**
	- Přihlašte se na počítači (vlastní složka je vytvořena prvním přihlášením)
	- a poté u své vlastní složky mají jenom ONI "čtení/zápis" a nikdo jiný tam není (kromě adminstrator.) a že jsou vlastníkem


- přepněte z "Práva uživetelů a skupin" na "přístup k hostitelům NFS"
	- zaškrtněte "Práva přístupu" u složky "servery"
	- dole u povolení přepněte "Možnosti povolení" z "pouze čtení" na "čtení/zápis"
