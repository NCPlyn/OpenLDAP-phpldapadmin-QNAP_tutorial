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
- Create a new **User Account** item under "users" OU, fill out everything, use /bin/bash as login shell.
  (If you want to have the user homes on remote NFS storage, make their home for example `/homenas/their-usrname`)
  Make sure their UID doesn't match any existing UID on the client machines

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
    - Create folder for the homes you specified while creating the User Account, ex.: `sudo mkdir /homenas`
    - `sudo nano /etc/fstab`
        - Add this to the end of your file with the right domain/IP of the server and folder, ex.: `nas.organiz.com:/servery /homenas nfs defaults 0 0`
    
- Reboot client machine
- For QNAP NAS:
    - Login into the account on the client machine (it will create it's homedir)
    - Login into the QNAP and open the permissions tab for their folder in /servery
    - **Remove any group or user that isn't the user who created the folder or administrator**

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