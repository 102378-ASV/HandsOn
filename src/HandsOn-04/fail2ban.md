# Fail2Ban

[Fail2Ban](https://github.com/fail2ban/fail2ban) és una eina de seguretat informàtica dissenyada per protegir els servidors contra intents d'intrusió i atacs automàtics. Aquesta aplicació monitoritza els registres de diverses aplicacions, com ara serveis de xarxa, per identificar comportaments sospitosos i bloquejar automàticament els usuaris o adreces IP que intenten accedir de manera repetida i no autoritzada al sistema.

El funcionament de **Fail2Ban** es basa en l'anàlisi dels registres del sistema en cerca de patrons o comportaments específics que indiquin intents d'intrusió. Quan es detecta un patró que coincideix amb les regles configurades, **Fail2Ban** pren accions preventives, com ara bloquejar temporalment l'adreça IP de l'origen. Aquesta funcionalitat ajuda a prevenir atacs d'enginyeria social, atacs de força bruta i altres activitats malintencionades.

* **Automatització de la seguretat**: Fail2Ban automatitza la detecció i resposta als intents d'intrusió, minimitzant la intervenció manual i augmentant l'eficiència en la gestió de la seguretat del sistema.
  
* **Reducció del risc d'intrusió**: Bloqueja automàticament les adreces IP que realitzen intents d'accés no autoritzats, disminuint el risc d'èxit d'atacs d'enginyeria social o de força bruta.
  
* **Flexibilitat de configuració**: Permet als administradors definir regles i filtres adaptats a les necessitats específiques del sistema i a les amenaces que es preveuen.
  
**Alertes i informes detallats**: Proporciona informació detallada sobre els intents d'intrusió, facilitant la identificació de patrons i millorant la resposta de seguretat.

## Instal·lació

1. Actualitza el sistema:

    ```bash
    dnf update -y
    ```

2. Instal·la Fail2Ban:

    ```bash
    dnf install fail2ban -y
    ```

3. Inicia i habilita el servei de Fail2Ban:

    ```bash
    systemctl enable --now fail2ban
    ```

## Configuració

Els fitxers de configuració de Fail2Ban es troben al directori **/etc/fail2ban/**. Els fitxers més importants són:

* **jail.conf** (arxiu de configuració principal)
* **jail.d/** (directori que conté fitxers de configuració addicionals per a serveis específics)
  
Per fer modificacions a la configuració, és recomanable copiar el fitxer de configuració del jail que vols modificar a jail.d/ per mantenir les modificacions separades de l'arxiu principal:

```bash
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.d/custom.conf
```

### Configuració dels filtres

Fail2Ban ofereix diverses accions que es poden configurar per manejar els intents d'intrusió. Algunes de les accions comunes inclouen:

* **banaction**: Determina quina acció prendre quan es detecta un intent d'intrusió (per defecte: iptables).
* **bantime**: Temps en què es mantindrà bloquejada l'adreça IP (per defecte: 10 minuts).
* **maxretry**: Número màxim d'intents permesos abans de bloquejar l'adreça IP (per defecte: 5).

Per configurar aquestes accions, pots modificar el fitxer *jail.local* o crear un nou fitxer a **jail.d/**:

```ini
[DEFAULT]
banaction = iptables-multiport
bantime = 3600
maxretry = 5
```

Recordeu de reiniciar el servei de Fail2Ban per aplicar els canvis:

```bash
systemctl restart fail2ban
```

### Creació de filtres pel servei SSH

En aquest punt crearem un filtre per protegir el servei SSH. Per fer-ho, primer cal crear un fitxer de configuració per al filtre SSH a **/etc/fail2ban/filter.d/sshd.conf**:

```ini
[Definition]
failregex = ^.*Failed password for .* from <HOST>.*$
ignoreregex =
```

Un cop definit el filtre podem revisar la seva configuració amb la comanda fail2ban-regex:

```bash
fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```

Si tot ha anat bé, hauríeu de veure una sortida similar a aquesta:

```bash
Running tests
=============
Use   failregex filter file : sshd, basedir: /etc/fail2ban
Use         log file : /var/log/auth.log
Use         encoding : UTF-8
```

Ara anem a configurar el jail per al servei SSH. Per fer-ho, crearem un fitxer de configuració a **/etc/fail2ban/jail.d/sshd.conf**:

```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
```

Recordeu de reiniciar el servei de Fail2Ban per aplicar els canvis:

```bash
systemctl restart fail2ban
```

## Simulant un intent d'intrusió (SSH)

Per provar el funcionament de Fail2Ban, podeu intentar accedir al servidor per SSH amb un usuari i contrasenya incorrectes. Si ho feu diverses vegades, l'adreça IP des de la qual esteu fent els intents d'intrusió serà bloquejada durant el temps especificat a la configuració.

Per fer aquesta simulació, crearem un usuari nou i li assignarem una contrasenya:

```bash
useradd -m sshuser
passwd sshuser
```

També caldrà modificar la configuració del servei SSH per permetre l'accés amb contrasenya:

```bash
vi /etc/ssh/sshd_config
```

Canvieu la línia *PasswordAuthentication* perquè quedi així:

```bash
PasswordAuthentication yes
```

Reinicia el servei SSH per aplicar els canvis:

```bash
systemctl restart sshd
```

Ara ja podeu provar d'accedir al servidor per SSH amb l'usuari i contrasenya que heu creat. Per tant, en un altre servidor, executeu la següent comanda:

```bash
ssh sshuser@<IP>
```

o bé podem fer un script automatitzat que faci un intent d'intrusió:

```bash
#!/bin/bash
TRIES=10
PASSWORD="badpassword"
for i in $(seq 1 $TRIES)
do
    echo $PASSWORD | ssh sshuser@<IP>
done
```

Un cop hagueu fet els intents d'intrusió, podeu comprovar que l'adreça IP des de la qual heu fet els intents d'intrusió ha estat bloquejada:

```bash
fail2ban-client status sshd
```

Per desbloquejar l'adreça IP, podeu executar la següent comanda:

```bash
fail2ban-client set sshd unbanip <IP>
```

## Manteniment

Pots revisar els registres de Fail2Ban per veure quines accions s'han pres i quines adreces IP s'han bloquejat. Els registres es troben generalment a */var/log/fail2ban.log*.

És també recomanable revisar periòdicament les configuracions de Fail2Ban per assegurar-se que estan actualitzades i són adequades per a les necessitats del sistema.

## Repte

### Notificacions per Correu Electrònic

Es pot configurar Fail2Ban per enviar notificacions per correu electrònic quan es detecten intents d'intrusió. Aquesta funcionalitat permet als administradors rebre alertes immediates sobre les accions preses per Fail2Ban, com ara bloqueigs d'adreces IP.

Per configurar les notificacions per correu electrònic, cal editar el fitxer de configuració de Fail2Ban:

```bash
vi /etc/fail2ban/jail.d/custom.conf
```

Afegiu les següents línies al fitxer:

```ini
[DEFAULT]
destemail = x@gmail.com  
sendername = Fail2Ban
mta = client_correu[postfix,sendmail,...]
```

Per tant, és necessari tenir un client de correu electrònic configurat en el teu servidor o màquina on està instal·lat Fail2Ban. Podeu fer servir [Postfix](https://www.postfix.org/) o [Sendmail](https://www.sendmail.com/).

Recordeu de reiniciar el servei de Fail2Ban per aplicar els canvis:

```bash
sudo systemctl restart fail2ban
```

Ara podeu tornar a fer un intent d'intrusió per comprovar que rebreu un correu electrònic amb la notificació.
