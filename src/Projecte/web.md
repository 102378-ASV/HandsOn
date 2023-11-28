# Servidor Web (mdbook)

En aquest apartat es descriu la configuració d'un servidor web basat en apache per a servir un llibre escrit en mdbook.

## Objectius

- Instal·lar i configurar un servidor web apache.
- Instal·lar i configurar un certificat SSL.
- Instal·lar i configurar mdbook.
- Fer una configuració segura del servidor web.
- Fer un desplegament automatitzat del llibre mdbook.

## Preparatius inicials

1. Crear una màquina virtual amb *Rocky Linux 8* amb el nom `mdbook-web` i amb 1GB de RAM, 0.25 CPU i 1 VCPU.

2. Configurar el nom de la màquina virtual:

    ```bash
    hostnamectl set-hostname mdbook.tc.udl.cat
    ```

3. Actualitzarem el sistema:

    ```bash
    dnf update -y
    ```

4. Instal·larem un servidor web apache com a servei web:

    ```bash
    dnf install -y httpd
    ```

5. Activarem i inciarem el servei d'Apache:

    ```bash
    systemctl enable --now httpd
    ```

6. Farem una còpia de seguretat de la configuració d'Apache:

    ```bash
    cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.original
    ```

7. Podeu crear un fitxer *index.html* per a comprovar que el vostre servidor web funciona correctament.

    ```bash
    cat << EOF > /var/www/html/index.html
    Hola món!
    EOF
    ```

## Configuració del tallafocs

1. Instal·larem el paquet `firewalld`:

    ```bash
    dnf install -y firewalld
    ```

2. Activarem i iniciarem el servei de `firewalld`:

    ```bash
    systemctl enable --now firewalld
    ```

3. Afegirem el servei web al tallafocs:

    ```bash
    firewall-cmd --add-service=http --permanent
    ```

4. Recarregarem la configuració del tallafocs:

    ```bash
    firewall-cmd --reload
    ```

**OBSERVACIÓ**: Aquest servidor web únicament tindrà obert el port 80 (http) i el port 22 (ssh) per la seva administració remota. Podeu comprovar-ho amb la comanda `firewall-cmd --list-all` o instal·lant el paquet `dnf install net-tools -y` i utilitzant `netstat -tulpn`.

## Certificat SSL

Es important que el nostre servidor web utilitzi un certificat SSL per a que les connexions siguin segures. En sistemes reals, podriau utilitzar el servei gratuït de [Let's Encrypt](https://letsencrypt.org/). Però en el nostre cas que treballem en una intranet, utilitzarem un certificat autofirmat.

1. Crearem un directori per a guardar els certificats:

    ```bash
    mkdir /etc/httpd/ssl
    ```

2. Generarem el certificat:

    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/httpd/ssl/apache.key -out /etc/httpd/ssl/apache.crt
    ```

    ```shell
    Generating a RSA private key
    ...
    Country Name (2 letter code) [XX]:ES
    State or Province Name (full name) []:Lleida
    Locality Name (eg, city) [Default City]:Lleida
    Organization Name (eg, company) [Default Company Ltd]:UDL
    Organizational Unit Name (eg, section) []:TC
    Common Name (eg, your name or your server's hostname) []:mdbook.tc.udl.cat
    Email Address []: tc@udl.cat
    ```

    Observeu que generem un certificat amb una durada de 365 dies. Utilitzem una clau RSA de 2048 bits i el guardem en el directori `/etc/httpd/ssl`.

3. Instal·larem el paquet `mod_ssl` per a que Apache pugui utilitzar el certificat:

    ```bash
    dnf install -y mod_ssl
    ```

4. Ara editarem `/etc/httpd/conf.d/ssl.conf` i afegirem el següent contingut:

    ```bash
    # Comenteu les linies següents i actualitzeu-les amb:
    SSLCertificateFile /etc/httpd/ssl/apache.crt
    SSLCertificateKeyFile /etc/httpd/ssl/apache.key
    ```

5. Reiniciarem el servei d'Apache:

    ```bash
    systemctl restart httpd
    ```

6. Obrirem el port 443 (https) al tallafocs:

    ```bash
    firewall-cmd --add-service=https --permanent
    firewall-cmd --reload
    ```

7. Reviseu que el vostre servidor web funciona correctament amb el protocol https.

    **NOTA**: Els navegadors per defecte no confiaran en els certificats autofirmats. Per aquest motiu, el navegador us mostrarà un missatge d'error. Per a evitar-ho, podeu instal·lar el certificat en el vostre navegador.

8. Ens pot interessar que els nostres usuari sempre utilitzin el protocol https, configurarem apache per redirigir totes les peticions al port 80 al port 443. Per fer-ho editarem el fitxer `/etc/httpd/conf.d/vhost.conf` i afegirem el següent contingut:

    ```bash
    <VirtualHost *:80>
        ServerName mdbook.tc.udl.cat
        Redirect permanent / https://192.168.101.X
    </VirtualHost>
    ```

    Reiniciarem el servei d'Apache:

    ```bash
    systemctl restart httpd
    ```

    I comprovarem que les peticions al port 80 són redirigides al port 443.

## Instal·lació de mdbook i git

1. Per instal·lar mdbook, primer necessitarem instal·lar el paquet `rust`:

    ```bash
    dnf install -y rust cargo  
    ```

2. Instal·larem el paquet `git`:

    ```bash
    dnf install -y git
    ```

## Crearem un usuari especial per a mdbook

Aquest usuari tindrà permisos per actualitzar el llibre mdbook que desplegarem amb apache a través de git i github. Però no podrà fer cap altra cosa.


1. Crearem un usuari amb el nom `mdbook` sense home directory:

    ```bash
    useradd -m mdbook
    ```

2. Crearem una clau ssh per a l'usuari `mdbook`:

    ```bash
    su - mdbook
    ssh-keygen -t ed25519 -C "mdbook.tc.udl.cat"

    # No cal que introduïu cap contrasenya.
    ```

3. Creareu un repositori: Podeu utiltizar aquest repositori template: [Repo]().

4. Afegirem la clau ssh al nostre repositori de github:

    ```bash
    cat /home/mdbook/.ssh/id_ed25519.pub
    # 1. Heu de copiar el contingut de la clau 
    # 2. Anar a settings del repositori del github
    # 3. Afegir-la a DEPLOY KEYS.
    # Tittle: mdbook.tc.udl.cat
    # Key: <copieu el contingut de la clau>
    # No marcar "Allow write access"
    ```

5. Crearem un directori per al source del nostre llibre:

    ```bash
    # Com a usuari root
    mkdir /mdbook-source
    chown mdbook:apache /mdbook-source
    ```

    **Nota**: Necessitem que l'owner sigui l'usuari encarregat de compilar el llibre i el grup sigui apache per a que el servidor web pugui accedir al directori.

6. Cloneu el repositori:

    ```bash
    # Com a usuari mdbook
    cd /mdbook-source
    git clone git@github.com:JordiMateoUdL/mdbook.git
    # Com a usuari root
    chown -R  mdbook:apache /mdbook-source
    ```

7. Instal·larem mdbook:

    ```bash
    cargo install --locked mdbook --vers 0.4.34
    ```

8. Afegireu el PATH de mdbook al PATH de l'usuari `mdbook`:

    ```bash
    echo 'export PATH=$PATH:/home/mdbook/.cargo/bin' >> /home/mdbook/.bashrc
    source /home/mdbook/.bashrc
    ```

9. Genereu el llibre:

    ```bash
    cd /mdbook-source/mdbook
    mdbook build
    ```

10. Per desplegar el llibre, necessitarem moure el contingut del directori `book` al directori `/var/www/html/`.

    - Crearem un enllaç simbòlic al directori `/var/www/html/mdbook`:

        ```bash
        ln -s  /mdbook-source/mdbook/book /var/www/html/
        ```

    - Actualitzarem SELinux per a que permeti a Apache accedir al directori:

        ```bash
        chcon -R -t httpd_sys_content_t /mdbook-source/mdbook/book 
        restorecon -R /mdbook-source/mdbook/book 
        ```

    - Ara actualitzarem el nostre servidor web per servir el nostre llibre aprofitant tota la configuració anterior del servidor web. En aquest cas, haurem de crear un nou fitxer de configuració per aquest llibre. Crearem el fitxer `/etc/httpd/conf.d/mdbook.conf` amb el següent contingut:

        ```bash
        <VirtualHost *:443>
            ServerName mdbook.tc.udl.cat
            DocumentRoot /var/www/html/book
            ErrorLog /var/log/httpd/mdbook-error.log
            CustomLog /var/log/httpd/mdbook-access.log combined
            SSLEngine on
            SSLCertificateFile /etc/httpd/ssl/apache.crt
            SSLCertificateKeyFile /etc/httpd/ssl/apache.key
        </VirtualHost>
        ```

    - Reiniciarem el servei d'Apache:

        ```bash
        systemctl restart httpd
        ```

    - Comprovarem que el nostre llibre es desplega correctament.

## Desplegament automatitzat del llibre

Com el servidor es troba en una intranet, no podem utilitzar cap servei de CI/CD com github actions o gitlab CI. Tampoc podem utiltizar els webhooks de github per a que el servidor web es desplegui automàticament. La solució que us proposo es utilitzar un cronjob per a que cada dia es comprovi si hi ha hagut algun canvi en el repositori i en cas afirmatiu, es compili el llibre i es desplegui al servidor web.

1. Crearem un fitxer `/home/mdbook/update.sh` amb el següent contingut:

    ```bash
    #!/bin/bash

    # Change to the root directory of your mdBook documentation
    cd /mdbook-source/mdbook

    # Pull changes from the Git repository
    git pull origin master  # Replace 'master' with your branch name

    # Build mdBook
    mdbook build
    ```

2. Donarem permisos d'execució al fitxer:

    ```bash
    chmod +x /home/mdbook/update.sh
    ```

3. Crearem un cronjob per a que s'executi cada dia a les 00:00:

    ```bash
    crontab -e
    ```

    I afegirem la següent línia:

    ```bash
    0 0 * * * /home/mdbook/update.sh
    ```