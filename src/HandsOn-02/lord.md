# Lord of the system

![](../HandsOn-02/figs/lord-of-the-ring-trilogy.png)

Tres anillos para los Reyes Elfos bajo el cielo.

Siete para los Señores Enanos en casas de piedra.

Nueve para los Hombres Mortales condenados a morir.

Uno para el Señor Oscuro, sobre el trono oscuro.

Un Anillo para gobernarlos a todos. Un anillo para encontrarlos,

un Anillo para atraerlos a todos y atarlos en las tinieblas

en la Tierra de Mordor donde se extienden las Sombras

## Objectius

* Aprendre a gestionar comptes en servidors UNIX/Linux.
* Familiaritzar-se amb els mecanismes de protecció i control d'usuaris.

## Preparant el servidor

1. Crear una MV amb el nom **middle-earth** amb el SO base Rocky Linux. **(1 CPU, 500MB de RAM, 4GB de disc)**.
2. Actualitzar totes les llibreries amb ```dnf update -y```.
3. Instal·lar un editor de text (vim o nano) amb ```dnf install vim -y```.
4. Instal·lar la shell tcsh amb ```dnf install tcsh -y```.
5. Actualitzar el nom de l'amfitrió amb: ```hostnamectl set-hostname middlearth.udl.cat```.

## Mostrant informació de benvinguda

El fitxer **/etc/motd** és l'arxiu on es guarda un missatge de benvinguda; normalment és un arxiu de text senzill que es mostra als usuaris quan inicien sessió. Pot contenir informació com la benvinguda al sistema, informació d'actualitat, polítiques de l'empresa, enllaços a recursos importants o qualsevol altra cosa que es consideri útil per als usuaris en el moment d'iniciar sessió.

Copieu el text següent al fitxer **/etc/motd**:

```txt
#                Bienvenido a la tierra media!                  #
#                            *  *  *                            #
#                         *  *  *  *  *                         #
#                      *  *  *  *  *  *  *                      #
#                      *  *  *  *  *  *  *                      #
#                      *  *  *  *  *  *  *                      #
#                         *  *  *  *  *                         #
#                            *  *  *                            #
# El hogar está atrás, el mundo por delante, y ha               #
# y muchos caminos que recorrer a través de la                  #
# sombras hasta el borde de la noche, hasta que                 #
# las estrellas estén encendidas                                #
#################################################################
```

```sh
~ vim /etc/motd 
# Afegiu la informació a l'arxiu.
~ exit
```

Tornem a iniciar sessió **SSH**. Per veure el nostre banner, després de fer *login*.

## Mostrant informació: Connexions remotes

L'arxiu **/etc/issue.net** és similar al missatge del dia (**/etc/motd**), però aquest s'utilitza per mostrar un missatge als usuaris abans que aquests s'autentiquin en un servidor mitjançant protocols com *SSH*. Aquest missatge normalment conté informació bàsica o una benvinguda als usuaris quan intenten connectar-se al servidor.

Per activar-ho:

1. Copieu el contingut del fitxer */etc/issue.net* a */etc/issue.net.default*. D'aquesta manera sempre mantindrem una còpia del fitxer original sense editar.
2. Copieu el següent text al fitxer */etc/issue.net*:

```txt
#################################################################
#                   _    _           _   _                      #
#                  / \  | | ___ _ __| |_| |                     #
#                 / _ \ | |/ _ \ '__| __| |                     #
#                / ___ \| |  __/ |  | |_|_|                     #
#               /_/   \_\_|\___|_|   \__(_)                     #
#                                                               #
#               You are entering into Mordor!                   #
#   Username has been noted and has been sent to the server     #
#                       administrator!                          #
#################################################################
```

Finalment, tornem a iniciar sessió **SSH**. Per veure el nostre banner.

```sh
~ cp /etc/issue.net /etc/issue.net.default
~ vim /etc/issue.net
# Copieu el text
# Guardar i sortir
```

Per podeu veure el banner, s'ha d'editar la configuració del servei SSH:

```sh
vim /etc/ssh/sshd_config
# Descomentar 
Banner /etc/issue.net
# Guardar i sortir
systemctl restart sshd
exit
```

## Creació de grups

La comanda **groupadd** s'utilitza per crear nous grups en un sistema Linux. Els grups són una manera de gestionar col·leccions d'usuaris, permetent una administració eficaç de permisos i accés als recursos del sistema. Els grups ajuden a organitzar i controlar els usuaris amb un mateix conjunt de permisos.

```sh
groupadd [opcions] nom_del_grup
```

### Crea 4 grups amb els següents GIDs

|**nom**|**GID**|
|---|---|
|hobbits| 600|
|elfs| 700|
|nans| 800|
|mags| 900|

```sh
groupadd -g 600 hobbits 
groupadd -g 700 elfs 
groupadd -g 800 nans 
groupadd -g 900 mags
```

## Creant els usuaris

La comanda *useradd* s'utilitza per crear nous usuaris en un sistema Linux. Cada usuari té un identificador únic (**UID**) i pot ser assignat a un o més grups. Aquesta comanda també pot crear els directoris inicials (home directories) pels usuaris.

### Crea els següents comptes

|**usuari**  |**UID**|**GID**|**nom**    |
|--------|---|---|-------|
|frodo   |601|600|Frodo  |
|gollum  |602|600|Smeagol|
|samwise |603|600|Samwise|
|legolas |701|700|Legolas|
|gimli   |801|800|Gimli  |
|gandalf |901|900|Gandalf|

```sh
useradd frodo -g 600 -u 601 -c "Frodo, portador de l'anell" -m -d /home/frodo 
~ useradd gollum -g 600 -u 602 -c "Smeagol, el cercador de l'anell" -m -d /home/smeagol 
~ useradd samwise -g 600 -u 603 -c "Samwise, l'amic fidel" -m -d /home/samwise 
~ useradd legolas -g 700 -u 701 -c "Legolas, el mestre de l'arc" -m -d /home/legolas 
~ useradd gimli -g 800 -u 801 -c "Gimli, el valent guerrer" -m -d /home/gimli
~ useradd gandalf -g 900 -u 901 -c "Gandalf, el mag" -m -d /home/gandalf
```

La *contrasenya* de tots els comptes ha de ser *Tolkien2LOR*.

```sh
~passwd frodo 
~passwd gollum 
~passwd samwise 
~passwd legolas 
~passwd gandalf 
```

## Actualitzant Usuaris

* Afegiu l'usuari **root** al grup dels **mags**.

```sh
~ usermod -aG mags root
```

o bé podeu fer servir (*gpasswd*):

```sh
~ gpasswd -a root mags
```

* L'usuari **gollum** vol tenir el seu *home* amb el nom **smeagol**.

```sh
~ usermod -d /home/smeagol -m gollum
```

* L'usuari **legolas** vol tenir per defecte la *shell tcsh*.

```sh
~ usermod -s /bin/tcsh legolas
```

* L'usuari **gimli** no vol tenir *contrasenya*.

```sh
~ passwd -d gimli
```

## Notificació de la comarca

Un cop s'han creat els usuaris entrarem al sistema com a **frodo** i enviarem un *mail* a la resta amb el següent missatge: **Benvinguts a la companyia, anem direcció mordor**.

```sh
dnf -y install postfix
dnf -y install mailx
```

Postfix és un programari de servidor de correu electrònic que té com a objectiu principal gestionar l'enviament, recepció i l'encaminament de correus electrònics en un entorn de servidor. És conegut per la seva eficiència, seguretat i flexibilitat, i és àmpliament utilitzat en servidors de correu electrònic en tot el món.

### Configuració del postfix

Editeu el fitxer */etc/postfix/main.cf*:

* **myhostname** = mail.middlearth.udl.cat
* **mydomain** = udl.cat
* **myorigin** = \$mydomain
* **inet_interfaces** = all
* **inet_protocols** = ipv4
* **mydestination** = \$myhostname, localhost.$mydomain, localhost, \$mydomain
* **mynetworks** = 127.0.0.0/8
* **home_mailbox** = Maildir/

Afegiu al final del fitxer */etc/postfix/main.cf*:

```conf
# Amaga el tipus o la versió del programari SMTP
smtpd_banner = $myhostname ESMTP

# Afegeix el següent al final
# Desactiva la comanda SMTP VRFY
disable_vrfy_command = yes

# Requereix la comanda HELO als amfitrions emissors
smtpd_helo_required = yes

# Límit de mida d'un correu electrònic
# Exemple a continuació significa límit de 10M bytes
message_size_limit = 10240000

# Configuracions SMTP-Auth
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname
smtpd_recipient_restrictions = permit_mynetworks, permit_auth_destination, permit_sasl_authenticated, reject
```

### Configurant el dimoni

* Arrancant el dimoni
  
```sh
~ systemctl enable --now postfix
```

* Preparant les bústies
  
```sh
echo 'export MAIL=$HOME/Maildir' >> /etc/profile.d/mail.sh
```

### Prova

```sh
[root@middlearth ~]# su - frodo
[gandalf@middlearth ~]$ mail
[frodo@middlearth ~]$ mail gandalf@localhost
Subject: Notificació de la comarca
Benvinguts a la companyia, anem direcció mordor.
.
EOT
[frodo@middlearth ~]$ exit
[root@middlearth ~]# su - gandalf
[gandalf@middlearth ~]$ mail
```

## Nasguls

* Creareu un usuari **nasgul** que pugui esdevenir **root**, però que no pugui accedir al sistema.

El grup *wheel* és un grup especial en alguns sistemes com els basats en Red Hat. Aquest grup té un significat històric i està relacionat amb la seguretat i els privilegis d'administració del sistema. En aquests sistmes, el grup **wheel** té permisos especials per accedir a determinades funcionalitats o comandes que requereixen permisos d'administrador o *root*. Aquest grup sol estar associat amb la possibilitat d'utilitzar la comanda **su** (superuser) per canviar a l'usuari *root* o altres usuaris amb privilegis d'administrador.

```sh
~ useradd nasgul -s /bin/nologin
~ usermod -aG wheel nasgul
```

* El **Frodo** ha sofert l’atac d’un **nasgul** i ha oblidat la seva *contrasenya*. Reinicialitza-la a **Hawkings** i assegura’t de què en el proper **login**, ell **l’actualitzarà**.

```sh
~ passwd -e Frodo
```

## Actualitzant l'equip

* Actualitza el *username* de **legolas** a **glorfindel**.

```sh
~ usermod legolas -l glorfindel
```

* Creació de fitxers i directoris amb l'usuari gimli:

```sh
su - gimli
touch espassa_nana.txt
mkdir tresors
exit
```

* Actualitza el *UID* de **gimli** a *800*.

```sh
usermod gimli -u 800
```

**Observació**: Quan es canvia l'UID, els fitxers i directoris creats anteriorment amb l'antic UID (en aquest cas, 801) ja no seran associats correctament a l'usuari gimli, ja que l'identificador ha canviat.

```sh
ls -la /home/gimli
```

Per solucionar-ho, podem seguir els passos següents:

```sh
find /home/gimli -user 801 -exec chown -h 800 {} +
```

Amb aquesta correcció, hem canviat l'UID de l'usuari gimli a 800 i també hem actualitzat el propietari dels fitxers i directoris associats amb l'antic UID (**801**) perquè ara estiguin associats correctament a l'usuari gimli amb el nou UID (**800**). 

* L'usuari **gandalf** ha de poder invocar a l'usuari **root**.

```sh
~ usermod -aG gandalf wheel
```

o bé:

```sh
~ gpasswd -a gandalf wheel
```

* Bloca el compte de **glorfindel**.

```sh
passwd -l glorfindel 
```

**OBSERVACIÓ:** Aquesta comanda no bloquejarà realment el compte, només canviarà la contrasenya per una contrasenya encriptada que no es pot desxifrar.

Per tant, la forma adequada de bloquejar el compte de l'usuari glorfindel i impedir que pugui iniciar sessió:

```sh
passwd -L glorfindel 
```

o bé (*chage*):

```sh
~ chage -E 0 glorfindel
```

## El poder de l'anell

* Crearem un directori **/anell**.

```sh
~ mkdir /anell
```

* Crearem un grup portadors.

```sh
~ groupadd portadors
```

* Assignarem a frodo com a propietari del directori **/anell**.

```sh
~ chown frodo:portadors /anell
```

* Modificarem els permisos del directori: Els fitxers d’aquest directori únicament podran ser **executats/editats** per l’usuari **Frodo**, la resta d’usuaris no ha de tenir cap permís ni de lectura, a excepció del grup d'usuari del grup **portadors** que *han de poder llegir el directori*.

```shl
~ chmod a-rwx /anell
~ chmod g+r /anell
~ chmod u+rwx /anell
```

* En **Frodo** ha de poder executar tots els fitxers del director **/anell/bin** sense haver d'afegir tota la ruta, únicament indicant el nom de l'executable.

```sh
~ echo "export PATH=$PATH:/anell/bin" >> $HOME/.bashrc 
~ source $HOME/.bashrc 
```

## Final del viatge

* Gimli es confon amb tots els missatges que apareixen a la pantalla quan inicia sessió. Configureu el seu compte perquè no es mostri cap missatge a la pantalla quan comenci la sessió.

```sh
~ su gimli
~ touch ~/.hushlogin
```

* No s’ha de permetre que en Gimli executi programes des del seu propi directori **/home**. Per fer-ho heu d'utilitzar (**setfacl**):

```sh
~ setfacl -m u:gimli:--- /home/gimli
```

* L’usuari Samwise s’ha perdut i ha acabat a Narnia, elimineu-lo de l’univers.

```sh
~userdel -r samwise
```

## Reptes

* Ajuda l'usuari *root* a configurar la shell **zsh** amb el framework [*Oh My Zsh*](https://ohmyz.sh/) i el tema [*Powerlevel10k*](https://github.com/romkatv/powerlevel10k). **Nota**: Aquesta configuració de l'intèrpret de comandes és la que utilitzo jo de forma habitual i és molt potent, configurable i flexible, per tant us la podeu configurar al vostres pc si voleu ^^.
