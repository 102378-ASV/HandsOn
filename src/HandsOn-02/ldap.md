# Gestió de comptes centralitzat amb LDAP

En aquest **HandsOn**, explorarem com implementar i configurar un servei d'autenticació centralitzada mitjançant [*LDAP*](https://www.openldap.org/doc/admin26/) en un servidor utilitzant el kernel de Linux. Aquesta tecnologia permet gestionar de manera centralitzada comptes d'usuari i grups, facilitant l'administració i el control d'accés als sistemes. 

![Objectiu del HandsOn](../HandsOn-02/figs/objectiu.png)

## Servidor LDAP

### Preprant el servidor

1. Podeu aprofitar la màquina virtual amb la base de dades de WordPress.
2. Actualitzeu totes les llibreries ```dnf update -y```.

### Instal·lació d'eines i dependències

Segons el [manual d'instal·lació](https://www.openldap.org/doc/admin26/install.html) necessitem instal·lar un conjunt d'eines:

```config
REQUIRED SOFTWARE
    Building OpenLDAP Software requires a number of software packages
    to be preinstalled.  Additional information regarding prerequisite
    software can be found in the OpenLDAP Administrator's Guide.

    Base system (libraries and tools):
        Standard C compiler (required)
        Cyrus SASL 2.1.27+ (recommended)
        OpenSSL 1.1.1+ (recommended)
        libevent 2.1.8+ (recommended)
        libargon2 or libsodium (recommended)
        Reentrant POSIX REGEX software (required)

    SLAPD:
        The ARGON2 password hashing module requires either libargon2
        or libsodium
    LLOADD:
        The LLOADD daemon or integrated slapd module requires
        libevent 2.1.8 or later.

    CLIENTS/CONTRIB ware:
        Depends on package.  See per package README.
```

* Instal·lació del compilador C:

```sh
~dnf install gcc -y
```

* Instal·lació de Cyrus SASL.:

```sh
~dnf install cyrus-sasl-devel  -y
```

* Instal·lació OpenSSL:

```sh
~dnf install openssl-devel -y
```

* Instal·lació de libevent:

```sh
~dnf install libevent-devel -y
```

* Instal·lació de libsodium:

```sh
~dnf install libsodium-devel -y
```

* Instal·lació de Software POSIX REGEX:

```sh
~dnf install pcre-devel -y
```

* Instal·lació de Perl:

```sh
~dnf install perl perl-devel -y
```

* **Eines de Configuració**: Adicionalment instal·larem eines que ens ajudin en la instal·lació i configuració.

```sh
dnf install make autoconf libtool vim tar wget -y
```

Finalment podem instal·lar extres com:

```sh
dnf install libtool-ltdl-devel libdb-devel -y
```

També podem afegir el repositori epel-release-7:

```sh
cat > /etc/yum.repos.d/epel-release-7.repo << 'EOF'
[epel-release-7]
name=Extra Packages for Enterprise Linux 7 - x86_64
baseurl=https://dl.fedoraproject.org/pub/epel/7/x86_64/
enabled=0
gpgcheck=0
EOF
```

Per poder instal·lar el extra wired-tiger.

```sh
dnf --enablerepo=epel-release-7 install wiredtiger wiredtiger-devel -y
```

**WiredTiger** és un sistema de gestió de bases de dades d'alt rendiment, escalable i de codi obert. És un motor de base de dades NoSQL i és conegut per la seva eficiència, velocitat i capacitat d'escala en aplicacions que requereixen un accés ràpid a la informació i manegar grans quantitats de dades.En el context d'OpenLDAP, WiredTiger és un dels motors de base de dades que es poden utilitzar per emmagatzemar les dades del directori LDAP.

### Creació d'un usuari/grup per gestionar el dimoni

Es una bona pràctica quan configurem servei tenir un usuari dedicat i un grup amb permisos restringits per executar aplicacions del sistema. Per tant, anem a fer-ho pel servei LDAP.

En primer lloc crearem el grup amb gid 55 anomenat ldap.

```sh
~ groupadd -g 55 ldap
```

L'usuari ldap no necessita directori al sistema i el seu directori personal el podem assignar a */var/lib/openldap*. El grup serà l'anterior amb gid 55 i podem assignar el uid 55 a l'usuari ldap. Finalment, podem impedir que aquest usuari inici sessió amb una shell *nologin*.

```sh
~ useradd -r -M -d /var/lib/openldap -u 55 -g 55 -s /usr/sbin/nologin ldap
```

### Descarregant el paquet d'instal·lació

```sh
cd /tmp
VER=2.6.6
wget ftp://ftp.openldap.org/pub/OpenLDAP/openldap-release/openldap-$VER.tgz
tar xzf openldap-$VER.tgz
cd openldap-$VER
```

### Configurant paquet

* Per veure totes les opcions de configuració:
  
```sh
./configure --help
```

* Configureu l'instal·lació amb les següents opcions:

```sh
# Configurant el paquet
./configure --prefix=/usr --sysconfdir=/etc --disable-static \
--enable-debug --with-tls=openssl --with-cyrus-sasl --enable-dynamic \
--enable-crypt --enable-spasswd --enable-slapd --enable-modules \
--enable-rlookups --enable-backends=mod --disable-sql \
--enable-ppolicy --enable-syslog --enable-overlays=mod
```

**Nota**: Si tot ha anat correctament, veure a la consola:

```consolse
# Please run "make depend" to build dependencies
```

Continuem construint les dependencies:

```sh
make depend
make
```

**Nota**: S'hauria de fer un make test per assegurar la correcta compilació, però requereix temps i ometrem el pas.

### Instal·lant el paquet

Per defecte, OpenLDAP utilitza l'algorisme de hash SHA-1 per emmagatzemar les contrasenyes. Aquesta configuració es pot trobar al fitxer de configuració d'OpenLDAP, generalment a la secció de configuració del mòdul de contrasenyes. Aquest modul és  considerat poc segur pel que fa a la seguretat de les contrasenyes, ja que s'han descobert vulnerabilitats que permeten atacs amb èxit. Es recomana utilitzar algorismes més forts com SHA-256, SHA-384 o SHA-512, que ofereixen una millor seguretat.

* Actualitzant a SHA-2:
  
```sh
cd contrib/slapd-modules/passwd/sha2
make
cd ../../../..
```

* Instal·lació d'OpenLDAP

```sh
make install
```

* Instal·lació de l'algorisme SHA-2:

```sh
cd contrib/slapd-modules/passwd/sha2
make install
```

* **Nota1**: Els fitxers de configuració d'OpenLDAP es guarden a **/etc/openldap**:

```sh
ls -la /etc/openldap/
```

* **Nota2**:  Les llibreries s'han instal·lat a **/usr/libexec/openldap**:

```sh
ls -la /usr/libexec/openldap
```

### Preparant la configuració


* Crearem un directori per guardar les dades **/var/lib/openldap**:

```sh
~ mkdir /var/lib/openldap
```

* Crearem un directori per la base de dades **/etc/openldap/slapd.d**:

```sh
~ mkdir /etc/openldap/slapd.d
```

* Assignem l'usuari i el grup *ldap* com a propietaris de  de **/var/lib/openldap**:

```sh
~ chown -R ldap:ldap /var/lib/openldap
```

* Assigneu a l'usuari root i al grup *ldap* com a propietaris de **etc/openldap/slapd.conf**:

```sh
~ chown root:ldap /etc/openldap/slapd.conf
```

* El fitxer  *etc/openldap/slapd.conf* ha de tenir els permissos *640*:

```sh
~ chmod 640 /etc/openldap/slapd.conf
```

### Creació del dimoni slapd

Per poder crear un servei (dimoni) necessitem un fitxer de configuració. 

En aquest fitxer establirem diferents seccions:

* En la primera secció **Unit** indicarem el dimoni del servidor OpenLDAP (*Description=OpenLDAP Server Daemon*) i que ha de ser iniciat després de **syslog.target** i **network-online.target**. També es proporcionen enllaços a la documentació corresponent (*Documentation=man:slapd* i *Documentation=man:slapd-mdb*). 
  
* En la segona secció **Service** s'estableix que el servei és de tipus **forking** (és a dir, es farà un ```fork()``` com a procés fill en segon pla), es defineix el fitxer de **PID** (*PIDFile=/var/lib/openldap/slapd.pid*) i es configuren les variables d'entorn per a les URL d'OpenLDAP i les opcions d'OpenLDAP. Finalment, s'especifica la comanda d'inici (**ExecStart**) per a iniciar el dimoni amb les opcions adequades. 

* Per acabar, l'última secció estableix la configuració d'instal·lació per al servei. Es diu que aquest servei és desitjat (**WantedBy**) per *multi-user.target*, el que significa que s'ha d'iniciar en l'arrencada del sistema quan es trobi en el mode *multi-usuari*.

```sh
cat > /etc/systemd/system/slapd.service << 'EOL'
[Unit]
Description=OpenLDAP Server Daemon
After=syslog.target network-online.target
Documentation=man:slapd
Documentation=man:slapd-mdb

[Service]
Type=forking
PIDFile=/var/lib/openldap/slapd.pid
Environment="SLAPD_URLS=ldap:/// ldapi:/// ldaps:///"
Environment="SLAPD_OPTIONS=-F /etc/openldap/slapd.d"
ExecStart=/usr/libexec/slapd -u ldap -g ldap -h ${SLAPD_URLS} $SLAPD_OPTIONS

[Install]
WantedBy=multi-user.target
EOL
```

### Preparació de la BD

* Crarem una còpia del fitxer original **/etc/openldap/slapd.ldif**:

```sh
~ mv /etc/openldap/slapd.ldif /etc/openldap/slapd.ldif.default
```

### Genració de contrasenyes amb  SHA-512

Per generar contrasenyes amb l'algorisme de hash SHA-512 (també conegut com a SSHA-512) utilitzant la comanda slappasswd, pots seguir aquests passos:

```sh
slappasswd -h "{SSHA512}" -o module-load=pw-sha2.la -o module-path=/usr/local/libexec/openldap
```

A continuació, la comanda demanarà la nova contrasenya:

```sh
# new password: 1234
# Re-enter new password: 1234
```

Un cop introdueixis la contrasenya, es generarà el hash corresponent amb l'algorisme SHA-512. El resultat hauria de ser similar a això:

```sh
{SSHA512}CBVaUdQC9mVvAi+0O92J3hA+aPdiWUqf4lVr6bGRAUsFJX5aFOEb+1pSsY8PQwW1UKuuCGO2+160HotnfjXIaRKlryVekLnu
```

Aquest és el hash de la contrasenya "1234" generat amb l'algorisme SHA-512.

**Nota**: D'ara en endavant...Si voleu modificar contrasenyes i utiltizar aquesta encriptació heu de seguir aquests passos.

### Creant la BD

El fitxer **slapd.ldif** és un fitxer d'entrada de dades en format LDIF (LDAP Data Interchange Format) que s'utilitza per a la configuració de la base de dades i altres paràmetres importants pel servidor OpenLDAP. En aquest fitxer s'estableix la configuració global, carrega mòduls, inclou els esquemes, configura les bases de dades, i defineix les polítiques d'accés a les dades. Aquesta configuració és crucial per al funcionament adequat del servidor OpenLDAP.

* **dn: cn=config**: Aquesta és la part del Distinguished Name (DN) que identifica la configuració global del servidor.
* **objectClass**: olcGlobal: Identifica l'objecte com una configuració global.
* **cn**: config: Indica el Common Name (CN) de l'objecte, que és "config" en aquest cas.
* **olcArgsFile**: L'arxiu on es guardaran els arguments per al servidor.
* **olcPidFile**: L'arxiu on es guardarà el PID (identificador de procés) del servidor.
* **olcTLSCipherSuite**: Configuració de les suites de xifrat per a TLS.
* **olcTLSProtocolMin**: La versió mínima del protocol TLS que s'accepta.

Es poden incloure [esquemes (schemes)](https://www.openldap.org/doc/admin22/schema.html) estàndard en el servidor OpenLDAP. Cada fitxer inclòs conté la definició de diferents classes d'objectes i atributs que es poden utilitzar. Com *core,cosine,nis o inetperson*.

```sh
cat > /etc/openldap/slapd.ldif << 'EOL'
dn: cn=config
objectClass: olcGlobal
cn: config
olcArgsFile: /var/lib/openldap/slapd.args
olcPidFile: /var/lib/openldap/slapd.pid
olcTLSCipherSuite: TLSv1.2:HIGH:!aNULL:!eNULL
olcTLSProtocolMin: 3.3

dn: cn=schema,cn=config
objectClass: olcSchemaConfig
cn: schema

dn: cn=module,cn=config
objectClass: olcModuleList
cn: module
olcModulepath: /usr/libexec/openldap
olcModuleload: back_mdb.la

dn: cn=module,cn=config
objectClass: olcModuleList
cn: module
olcModulepath: /usr/local/libexec/openldap
olcModuleload: pw-sha2.la

include: file:///etc/openldap/schema/core.ldif
include: file:///etc/openldap/schema/cosine.ldif
include: file:///etc/openldap/schema/nis.ldif
include: file:///etc/openldap/schema/inetorgperson.ldif

dn: olcDatabase=frontend,cn=config
objectClass: olcDatabaseConfig
objectClass: olcFrontendConfig
olcDatabase: frontend
olcPasswordHash: {SSHA512}CBVaUdQC9mVvAi+0O92J3hA+aPdiWUqf4lVr6bGRAUsFJX5aFOEb+1pSsY8PQwW1UKuuCGO2+160HotnfjXIaRKlryVekLnu
olcAccess: to dn.base="cn=Subschema" by * read
olcAccess: to * 
  by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage 
  by * none

dn: olcDatabase=config,cn=config
objectClass: olcDatabaseConfig
olcDatabase: config
olcRootDN: cn=config
olcAccess: to * 
  by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage 
  by * none
EOL
```

**Nota**: *back_mdb.la* és el mòdul que permet utilitzar la base de dades MDB (Memory-Mapped Database), i *pw-sha2.la* és un mòdul per al xifrat de contrasenyes amb l'algorisme SHA-2.

```sh
cd /etc/openldap/
slapadd -n 0 -F /etc/openldap/slapd.d -l /etc/openldap/slapd.ldif
```

```sh
ls /etc/openldap/slapd.d
# 'cn=config'  'cn=config.ldif'
```

```sh
chown -R ldap:ldap /etc/openldap/slapd.d
```

### Executant el dimoni

```sh
systemctl daemon-reload
systemctl enable --now slapd
systemctl status slapd
```

### Activant els logs

* Creació d'un fitxer per configura els logs:

```sh
cat > enable-openldap-log.ldif << 'EOL'
dn: cn=config
changeType: modify
replace: olcLogLevel
olcLogLevel: stats
EOL
```

* Afegim la configuració al servei:

```sh
ldapmodify -Y external -H ldapi:/// -f enable-openldap-log.ldif
#SASL/EXTERNAL authentication started
#SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
#SASL SSF: 0
#modifying entry "cn=config"
```

* Configurem Rsyslog:

```sh
cd ~
echo "local4.* /var/log/slapd.log" >> /etc/rsyslog.conf
```

* Reiniciem Rsyslog:

```sh
systemctl restart rsyslog
```

### Configurant estructura de la BD

```sh
cat > rootdn.ldif << 'EOL'
dn: olcDatabase=mdb,cn=config
objectClass: olcDatabaseConfig
objectClass: olcMdbConfig
olcDatabase: mdb
olcDbMaxSize: 42949672960
olcDbDirectory: /var/lib/openldap
olcSuffix: dc=curs,dc=asv,dc=udl,dc=cat
olcRootDN: cn=admin,dc=curs,dc=asv,dc=udl,dc=cat
olcRootPW: {SSHA512}CBVaUdQC9mVvAi+0O92J3hA+aPdiWUqf4lVr6bGRAUsFJX5aFOEb+1pSsY8PQwW1UKuuCGO2+160HotnfjXIaRKlryVekLnu
olcDbIndex: uid pres,eq
olcDbIndex: cn,sn pres,eq,approx,sub
olcDbIndex: mail pres,eq,sub
olcDbIndex: objectClass pres,eq
olcDbIndex: loginShell pres,eq
olcAccess: to attrs=userPassword,shadowLastChange,shadowExpire
  by self write
  by anonymous auth
  by dn.subtree="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage
  by dn.subtree="ou=system,dc=curs,dc=asv,dc=udl,dc=cat" read
  by * none
olcAccess: to dn.subtree="ou=systemdc=curs,dc=asv,dc=udl,dc=cat" 
  by dn.subtree="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage
  by * none
olcAccess: to dn.subtree="dc=curs,dc=asv,dc=udl,dc=cat" 
  by dn.subtree="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage
  by users read
  by * none
EOL

ldapadd -Y EXTERNAL -H ldapi:/// -f rootdn.ldif
```

En aquest fitxer estem configurant un usuari admin. Aquest usuari té privilegis elevats i pot realitzar diverses operacions d'administració en el servidor LDAP. Observeu les regles (*oclAcces*). Per defecte, s'utilitza *1234* com a contrasenya, per modificar-la actualitzeu el hash.

### Configurant communicacions segures SSL/TLS

Configurar comunicacions segures mitjançant SSL/TLS en un servidor LDAP és una pràctica important per garantir que les dades que es transmeten entre el client i el servidor estiguin xifrades i siguin segures.

* Xifrem les comunicacions amb el servidor d'autenticació:

```sh
mkdir /pki
openssl req -days 500 -newkey rsa:4096 \
    -keyout /pki/ldapkey.pem -nodes \
    -sha256 -x509 -out /pki/ldapcert.pem
```

* Actualitzem permissos:

```sh
chown ldap:ldap /pki/ldapkey.pem
chmod 400 /pki/ldapkey.pem
cat /pki/ldapcert.pem >> /pki/cacerts.pem
```

* Creació del fitxer de configuració:

```sh
cat > add-tls.ldif << 'EOL'
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /pki/cacerts.pem
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /pki/ldapkey.pem
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /pki/ldapcert.pem
EOL
```

* Actualitzem LDAP:
  
 ```sh
ldapadd -Y EXTERNAL -H ldapi:/// -f add-tls.ldif
```

* Actualitzem la configuració:

```sh
vim /etc/openldap/ldap.conf
#... editar...
#TLS_CACERT     /etc/pki/tls/cert.pem
TLS_CACERT     /pki/ldapcert.pem
```

### Organitzant la jerarquia

Ara crearem una organització semblant a la de linux amb les següents entrades:

* **dc=curs,dc=asv,dc=udl,dc=cat**: Node principal de la jerarquia LDAP.
* **ou=groups,dc=curs,dc=asv,dc=udl,dc=cat**.
* **ou=users,dc=curs,dc=asv,dc=udl,dc=cat**.
* **ou=system,dc=curs,dc=asv,dc=udl,dc=cat**:Aquesta entrada representa una organització per al sistema. És comuna en moltes configuracions LDAP. També utilitza les mateixes classes que les altres organitzacions, però la propietat *ou* té el valor *system*.

```ldif
cat > basedn.ldif << 'EOL'
dn: dc=curs,dc=asv,dc=udl,dc=cat
objectClass: dcObject
objectClass: organization
objectClass: top
o: Curs ASV
dc: curs

dn: ou=groups,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: organizationalUnit
objectClass: top
ou: groups

dn: ou=users,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: organizationalUnit
objectClass: top
ou: users

dn: ou=system,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: organizationalUnit
objectClass: top
ou: system
EOL
```

```sh
ldapadd -Y EXTERNAL -H ldapi:/// -f basedn.ldif
```

### Creant grups i usuaris


Per crear un usuari dins la jerarquia que has definit, necessitaràs crear una nova entrada d'usuari amb les propietats adequades. Aquí tens un exemple de com fer-ho en format LDIF:

```ldif
dn: uid=johndoe,ou=users,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
objectClass: top
cn: John Doe
sn: Doe
uid: johndoe
uidNumber: 1000
gidNumber: 1000
homeDirectory: /home/johndoe
loginShell: /bin/bash
userPassword: {SSHA512}CBVaUdQC9mVvAi+0O92J3hA+aPdiWUqf4lVr6bGRAUsFJX5aFOEb+1pSsY8PQwW1UKuuCGO2+160HotnfjXIaRKlryVekLnu
```

on:

* **dn**: Distinquished Name (DN) de l'usuari, que indica la seva ubicació dins de la jerarquia. En aquest cas, està dins de la branca "ou=users,dc=curs,dc=asv,dc=udl,dc=cat".
* **objectClass**: Indica les classes d'objectes a les quals pertany aquesta entrada. En aquest cas, pertany a les classes inetOrgPerson, posixAccount, shadowAccount i top, que defineixen les propietats i característiques de l'usuari.
* **cn**: El nom complert de l'usuari, en aquest cas "John Doe".
* **sn**: El cognom de l'usuari, en aquest cas "Doe".
* **uid**: L'identificador únic de l'usuari.
* **uidNumber**: L'identificador únic de l'usuari en termes numèrics.
* **gidNumber**: L'identificador únic del grup al qual pertany l'usuari en termes numèrics.
* **homeDirectory**: La carpeta d'inici de l'usuari.
* **loginShell**: L'intèrpret de comandes que utilitzarà l'usuari en iniciar sessió.
* **userPassword**: El hash de la contrasenya de l'usuari (en aquest cas, utilitzant l'algorisme SHA-512). Recordeu que aquesta és una versió encriptada de la contrasenya "1234" generada prèviament amb l'algorisme SHA-512. En la pràctica, caldria utilitzar un hash de la contrasenya de l'usuari que estigui segur.

Ara crearem un usuari **OSProxy**. Això és una bona pràctica per a un sistema amb l'objectiu de resoldre UIDs (Identificadors d'Usuari) i GIDs (Identificadors de Grup) de manera centralitzada i eficient. El objectiu es permetre que altres màquines puguin connectar-se a aquest servidor per resoldre el seus usuaris. Tenir aquest usuari ens permetra:

* **Separació de Privilegis**: L'usuari OSProxy té funcions específiques per a la resolució d'UIDs i GIDs. Aquesta separació de privilegis és una pràctica de seguretat que redueix l'abast de les possibles vulnerabilitats.

* **Control d'Accés**: L'usuari OSProxy pot tenir permisos limitats només per a la resolució d'UIDs i GIDs, i no per a altres operacions crítiques de l'LDAP. Això millora la seguretat de la base de dades LDAP restringint l'accés només al que és necessari.

* **Escalabilitat i Rendiment**: L'ús d'un usuari dedicat per a aquesta funcionalitat específica pot optimitzar les operacions de consulta d'UIDs i GIDs, garantint un rendiment més alt i una resposta més ràpida del sistema.

```ldif
dn: cn=osproxy,ou=system,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: osproxy
userPassword:{SSHA512}CBVaUdQC9mVvAi+0O92J3hA+aPdiWUqf4lVr6bGRAUsFJX5aFOEb+1pSsY8PQwW1UKuuCGO2+160HotnfjXIaRKlryVekLnu
description: OS proxy for resolving UIDs/GIDs
```

També podem aprofitar per crear grups i usuaris:


```ldif
dn: cn=programadors,ou=groups,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: posixGroup
cn: programadors
gidNumber: 5000
memberUid: jordi

dn: cn=dissenyadors,ou=groups,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: posixGroup
cn: programadors
gidNumber: 5001
memberUid: manel

dn: uid=jordi,ou=users,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
cn: Jordi
sn: Mateo
uid: jordi
uidNumber: 5000
gidNumber: 5000
homeDirectory: /home/jordi
loginShell: /bin/sh
gecos: Full Name

dn: uid=manel,ou=users,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
cn: manel
sn: Mateo
uid: manel
uidNumber: 5001
gidNumber: 5001
homeDirectory: /home/manel
loginShell: /bin/sh
gecos: Full Name
```

En definitiva:

```sh
cat > users.ldif << 'EOL'
dn: cn=osproxy,ou=system,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: osproxy
userPassword:{SSHA512}CBVaUdQC9mVvAi+0O92J3hA+aPdiWUqf4lVr6bGRAUsFJX5aFOEb+1pSsY8PQwW1UKuuCGO2+160HotnfjXIaRKlryVekLnu
description: OS proxy for resolving UIDs/GIDs

dn: cn=programadors,ou=groups,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: posixGroup
cn: programadors
gidNumber: 5000
memberUid: jordi

dn: cn=dissenyadors,ou=groups,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: posixGroup
cn: programadors
gidNumber: 5001
memberUid: manel

dn: uid=jordi,ou=users,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
cn: Jordi
sn: Mateo
uid: jordi
uidNumber: 4000
gidNumber: 5000
homeDirectory: /home/jordi
loginShell: /bin/sh
gecos: Full Name

dn: uid=manel,ou=users,dc=curs,dc=asv,dc=udl,dc=cat
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
cn: manel
sn: Mateo
uid: manel
uidNumber: 4001
gidNumber: 5001
homeDirectory: /home/manel
loginShell: /bin/sh
gecos: Full Name
EOL
```

* Afegim els usuaris i grups al sistema:
  
```sh
ldapadd -Y EXTERNAL -H ldapi:/// -f users.ldif
``````

### Inicialitzant contrasenyes

```sh
ldappasswd -H ldapi:/// -Y EXTERNAL -S "uid=jordi,ou=users,dc=curs,dc=asv,dc=udl,dc=cat"
# 1234
ldappasswd -H ldapi:/// -Y EXTERNAL -S "uid=manel,ou=users,dc=curs,dc=asv,dc=udl,dc=cat"
# 1234
ldappasswd -H ldapi:/// -Y EXTERNAL -S "cn=osproxy,ou=system,dc=curs,dc=asv,dc=udl,dc=cat"
# 1234
```

### Cerques a LDAP

```sh
ldapsearch -x -W -H ldapi:/// \
 -D "uid=manel,ou=users,dc=curs,dc=asv,dc=udl,dc=cat" \
 -b "ou=users,dc=curs,dc=asv,dc=udl,dc=cat"
```

### Firewall

**OBSERVACIÓ**: Si esteu fent servir una màquina ja configurada, és possible que el firewall estigui activat. Haureu d'obrir únicament el servei LDAP.

```sh
dnf install firewalld -y
systemctl start --now firewalld
```

* Permetre connexions remotes **LDAP (389 UDP/TCP)** i **LDAPS (636 UDP/TCP)**.
  
```sh
firewall-cmd --add-service={ldap,ldaps} --permanent
firewall-cmd --reload
```

## Client

### Preparant el servidor

* Podeu utlitzar una de les repliques de Wordpress.
* Si configureu una nova màquin: **ldap-client** amb el SO base Rocky Linux. **(1 CPU, 1GB de RAM, 4GB de disc)**.

#### Al client (ldap-client)

1. Genereu una clau ssh: ```ssh-keygen -t ed25519```.
2. Copieu la clau pública del client al servidor LDAP: ```cat .ssh/id_ed25519.pub```.

#### Al Servidor (ldap-server)

```sh
vim /root/.ssh/authorized_keys
# Enganxem la clau
systemctl restart sshd
```

## Instal·lació dels paquets necessaris

```sh
dnf -y install openldap-clients sssd sssd-tools oddjob-mkhomedir vim
```

## Configuració del DNS

```sh
vim /etc/hosts
# Afegiu al final:
192.168.101.84 curs.asv.udl.cat
# On
#ip_del_vostre_servidor_ldap
```

## Configurant certificats SSL/TLS

```sh
mkdir -p /etc/pki/tls
cd /etc/pki/tls
sftp root@curs.asv.udl.cat:/pki
    #The authenticity of host '192.168.101.84 (192.168.101.84)' 
    #can't be established.
    #ECDSA key fingerprint is SHA256:iINseqop81faj9v6AZRPFX8hvri+IA6yGODAG97+XBY.
    #Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
    #Warning: Permanently added '192.168.101.84' (ECDSA) to the list 
    # of known hosts.
    #Connected to 192.168.101.84.
    #Changing to: /pki
sftp> get cacerts.pem
    #Fetching /pki/cacerts.pem to cacerts.pem
    #/pki/cacerts.pem                         100% 2143   574.9KB/s   00:00
sftp> exit
mv cacerts.pem cacerts.crt
```

## Configuració SSSD -> LDAP

El daemon SSSD (System Security Services Daemon), proporciona integració amb serveis d'autenticació i autorització en sistemes Linux. Les parts principals de la seva configuració (**/etc/sssd/sssd.conf**) són:

* **[sssd]**: Secció principal on es configuren els serveis que s'han d'activar.
  * **services**: Especifica els serveis actius, com nss (informació de noms), pam (autenticació) i sudo (privilegis d'administrador).
  * **config_file_version**: La versió de la configuració.
  * **domains**: Els dominis amb configuracions específiques.
* **[sudo], [nss], [pam]**: Seccions per a la configuració específica de cada servei. Per exemple, la configuració de les credencials en mode "offline" per a PAM.
* **[domain/default]**: Configuració del domini per defecte.
  * *ldap_id_use_start_tls*: Usa TLS per a les connexions LDAP.
  * *cache_credentials*: Emmagatzema les credencials en memòria.
  * *ldap_search_base*: Base de cerca a LDAP.
  * *id_provider, auth_provider, chpass_provider*: Proveïdors per a la identificació, autenticació i canvi de contrasenya (LDAP en aquest cas).
  * *ldap_uri*: URI del servidor LDAP.
  * *ldap_default_bind_dn*: L'identificador per defecte per a les connexions LDAP.
  * *ldap_group_search_base*, ldap_user_search_base: Bases de cerca per a grups i usuaris.
  * *ldap_default_authtok*: La contrasenya encriptada per a la connexió LDAP.
  * *ldap_tls_reqcert, ldap_tls_cacert, ldap_tls_cacertdir*: Configuració de TLS.
  * *ldap_search_timeout, ldap_network_timeout*: Temps d'espera per a cerques i xarxa.
  * *ldap_access_order, ldap_access_filter*: Ordre i filtre per a l'accés LDAP.

```sh
cat << EOL >> /etc/sssd/sssd.conf
[sssd]
services = nss, pam, sudo
config_file_version = 2
domains = default

[sudo]

[nss]

[pam]
offline_credentials_expiration = 60

[domain/default]
ldap_id_use_start_tls = True
cache_credentials = True
ldap_search_base = dc=curs,dc=asv,dc=udl,dc=cat
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap
access_provider = ldap
sudo_provider = ldap
ldap_uri = ldaps://ldap.asv.udl.cat
ldap_default_bind_dn = cn=osproxy,ou=system, dc=curs,dc=asv,dc=udl,dc=cat
ldap_group_search_base = ou=groups,dc=curs,dc=asv,dc=udl,dc=cat
ldap_user_search_base = ou=users,dc=curs,dc=asv,dc=udl,dc=cat
ldap_default_authtok = 1234
ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/pki/tls/cacert.crt
ldap_tls_cacertdir = /etc/pki/tls
ldap_search_timeout = 50
ldap_network_timeout = 60
ldap_access_order = filter
ldap_access_filter = (objectClass=posixAccount)
EOL
```

* Modifiqueu **/etc/openldap/ldap.conf**:
  
```sh
vim /etc/openldap/ldap.conf
# Modificacions
URI ldaps://curs.asv.udl.cat
BASE  dc=curs,dc=asv,dc=udl,dc=cat
TLS_CACERT      /etc/pki/tls/cacert.crt
```

### Configurant NSS i PAM per fer servir SSSD

```sh
authselect select sssd --force
```

### Configuració per la creació automàtica dels homes

* Activem el módul:

```sh
systemctl enable --now oddjobd
```

* Carregueu el mòdul pam_oddjob_mkhomedir al fitxer d'autenticació PAM **/etc/pam.d/system-auth** per habilitar la creació automàtica del directori d'inici:

```sh
echo "session optional pam_oddjob_mkhomedir.so skel=/etc/skel/ umask=0022" >> /etc/pam.d/system-auth 
```

* Reiniciem el dimoni:

```sh
systemctl restart oddjobd
```

### Executant el dimoni SSSD

* Revisem que la configuració sigui correcta:

```sh
sssctl config-check
```

* Atorguem els permissos corresponents a **/etc/sssd**:

```sh
chown -R root: /etc/sssd
chmod 600 -R /etc/sssd
```

* Activem el dimoni *sssd*:

```sh
systemctl enable --now sssd
```

## Prova

Per testar la configuració del client crearem un directori */home/jordi* i l'assignarem a un usuari i a un grup inicialitzat a *LDAP*.

```sh
mkdir /home/jordi
chown 4000:5000 /home/jordi
```

Si revisem les propietats veurem que el sistema detecta l'usuari amb jordi i programadors que són els **uid** i **gid** del usuari amb 4000 i 5000 definits a *LDAP*.

```sh
ls -l /home
drwxr-xr-x.  2 jordi programadors   6 Sep 26 18:36 jordi
```

## Solució de problemes

### Servidors

* Revisió de logs: ```journalctl -u slapd```
* Reinicia el servei:
  
  ```sh
    systemctl stop slapd
    cd /etc/openldap
    rm -rf slapd.d/
    chown -R ldap:ldap /etc/openldap/slapd.d
  ```

### Clients

* Revisió de logs: ```journalctl -u sssd```
* Reconfiguración de la Autenticación:

  ```sh
    authconfig --disablesssd --disablesssdauth --update
    systemctl restart sssd
  ```

## Reptes

* Configureu un Client LAM (Ldap Account Manager) a la màquina del servidor per poder gestionar els usuaris amb un client web. Seguiu les instruccions de [https://www.ldap-account-manager.org/lamcms/](https://www.ldap-account-manager.org/lamcms/).
* Configureu una màquina, un client, que utilitzi el servidor d'autenticació centralitzada LDAP, on els usuaris que configureu puguin esdevenir root. Aquesta configuració s'ha de fer a la base de dades *LDAP*, de manera que aquests usuaris puguin esdevenir root en qualsevol màquina de l'organització.
* Implementeu un script d'automatització per instal·lar el client i el servidor.
* Configureu el servidor *LDAP* per tenir els usuaris i els permisos del Lord of the system i connecteu-los amb el client de la màquina (lord-of-the-system).
