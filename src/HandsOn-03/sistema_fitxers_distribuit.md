# Sistema de fitxers distribuït

Un sistema de fitxers distribuït (DFS) permet l'accés i la gestió de fitxers de manera distribuïda en diversos nodes o màquines a través de la xarxa.

Hi ha diversos sistemes de fitxers distribuïts disponibles per a Linux. Alguns dels més populars inclouen:

* **[GlusterFS](https://www.gluster.org/)**: un sistema de fitxers distribuït d'alt rendiment i alta disponibilitat.
* **[Ceph](https://ceph.io/en/)**: un sistema de fitxers distribuït altament escalable i segur.
* **NFS**: un sistema de fitxers distribuït tradicional que s'utilitza des de fa molts anys.

En un sistema de fitxers distribuït, cada servidor té un inode per a cada fitxer o directori que comparteix. L'inode conté informació sobre el fitxer o directori, com ara el seu nom, el seu tamany, els permisos d'accés i el seu contingut.

El kernel de Linux gestiona l'accés als fitxers distribuïts. Quan un client NFS accedeix a un fitxer distribuït, el kernel de Linux envia una sol·licitud al servidor NFS. El servidor NFS envia la informació sobre el fitxer al kernel de Linux, que la utilitza per crear una representació local del fitxer.

El kernel de Linux també gestiona les modificacions dels fitxers distribuïts. Quan un client (DFS)modifica un fitxer distribuït, el kernel de Linux envia la modificació al servidor (DFS). El servidor (DFS) aplica la modificació al fitxer distribuït i envia una confirmació al kernel de Linux.

1. El Client (DFS) realitza una sol·licitud d'accés a un fitxer.
2. El Client espera la resposta amb informació sobre el fitxer.
3. El Servidor retorna la informació sobre el fitxer.
4. El Client fa una sol·licitud de modificació del fitxer.
5. El Servidor contesta amb la confirmació/denegació de la modificació.
6. El Client processa la resposta.

Els avantatges d'un sistema de fitxers distribuït en servidors Linux inclouen:

* **Escalabilitat**: el sistema de fitxers es pot escalar fàcilment afegint més servidors.
* **Disponibilitat**: el sistema de fitxers continua sent accessible fins i tot si un o més servidors fallen.
* **Seguretat**: els fitxers es poden protegir mitjançant accés i encriptació.

Els desavantatges d'un sistema de fitxers distribuït en servidors Linux inclouen:

* **Complexitat**: la configuració i la gestió d'un sistema de fitxers distribuït poden ser complexes.
* **Cost**: la implementació d'un sistema de fitxers distribuït pot ser costosa.
  
## NFS

**NFS** (Network File System) és un protocol que permet l'accés a sistemes de fitxers distribuïts en xarxes. Aquest protocol permet als usuaris i als sistemes accedir i compartir fitxers a través de la xarxa com si estiguessin emmagatzemats localment.

* **Servidor NFS**: Un servidor NFS és un sistema que té un conjunt de fitxers compartits que pot oferir a altres sistemes a través de la xarxa mitjançant el protocol NFS.

* **Client NFS**: Un client NFS és un sistema que accedeix als fitxers compartits en un servidor NFS mitjançant la xarxa utilitzant el protocol NFS.

* **Punt de Muntatge**: És un directori en el sistema de fitxers del client on es fa disponible el contingut del sistema de fitxers compartit del servidor.

* **Exports**: són directoris o sistemes de fitxers que es defineixen per ser compartits.

### Servidor NFS

#### Requísits previs

En aquest punt, assumirem que teniu una màquina virtual creada amb un sistema operatiu basat en Red Hat (com Rocky Linux) i que aquesta màquina té un disc virtual diferent del disc on està instal·lat el sistema operatiu. Per exemple, podem tenir un disc virtual de **500MB** que es munta a **/mnt/data**. Assumirem que en aquest servidor tenim activat *SELinux* i el firewall *firewalld*.

#### Instal·lació i Configuració del servidor NFS

1. Instal·lació dels paquets necessaris a Rocky Linux 8:

    ```sh
    dnf install nfs-utils -y
    ```

2. Habilitació dels serveis

    ```sh
    systemctl enable --now nfs-server
    ```

3. Configuració dels exports de NFS:

    ```sh
    echo "/mnt/data *(rw,sync,no_root_squash)" >> /etc/exports
    ```

    En aquesta configuració estem exportant el directori **/mnt/data** a tota la xarxa (*) i permetent l'accés en escriptura (**rw**) de forma síncrona (**sync**), sense restringir l'accés del root (**no_root_squash**).

4. Exportem la configuració:

    ```sh
    exportfs -avr
    ```

5. Configurem SELinux per permetre compartir el servei (**nfs_export_all_ro**) i permetem que el servei pugui llegir i escriure (**nfs_export_all_rw**).

    ```sh
    setsebool -P nfs_export_all_ro=1 nfs_export_all_rw=1
    ```

6. Configuració del firewall per permetre NFS: Obrirem el port utilitzat per NFS (normalment el port 2049) per permetre l'accés a través del firewall:

    ```sh
    firewall-cmd --permanent --add-service=nfs
    firewall-cmd --reload
    ```

7. Reiniciem els serveis

    ```sh
    systemctl restart rpcbind
    systemctl restart nfs
    ```

### Client NFS

En aquest punt, assumirem que teniu una màquina virtual creada amb un sistema operatiu basat en Red Hat (com Rocky Linux). Aquesta màquina ha d'estar a la mateixa xarxa que la màquina **Servidor NFS**.

#### Instal·lació i Configuració del client NFS

1. Instal·lació dels paquets NFS:

    ```sh
    dnf install nfs-utils -y
    ```

2. Iniciem el servei RPCbind:

    ```sh
    systemctl enable --now rcpbind
    ```

3. Crearem un punt de muntatge (**/mnt/data**):

    ```sh
    mkdir /mnt/data
    ```

4. Muntatge del sistema de fitxers NFS: Establirem que el sistema de fitxers es munti amb permisos de lectura i escriptura.

    ```sh
    # ip-servidor s'ha de substituir per la ip del servidor
    mount -t nfs -o rw ip-servidor:/mnt/data/ /mnt/data
    ```

5. Muntatge automàtic:

    ```sh
    # ip-servidor s'ha de substituir per la ip del servidor
    echo "ip-servidor:/mnt/data/ /mnt/data nfs rw,sync,hard,intr 0 0" >> /etc/fstab
    ```

    * **sync**: Aquesta opció especifica que les operacions d'escriptura seran sincròniques. Això vol dir que l'escriptura al sistema de fitxers NFS s'esperarà fins que es confirmi que s'ha completat amb èxit abans de continuar amb altres operacions. Això assegura una escriptura segura, però pot afectar el rendiment.

    * **hard**: Aquesta opció especifica que es faran intents d'accedir i reintentarà l'accés al servidor NFS si hi ha un error. Si el servidor NFS es torna inaccessible, l'operació d'E/S es quedarà en espera i intentarà de nou de manera persistent.
  
    * **intr**: Aquesta opció permet interrompre les operacions d'E/S que estan bloquejades en un sistema de fitxers NFS si el servidor esdevé inactiu. Sense aquesta opció, si el servidor NFS es torna inactiu, les operacions d'E/S podrien quedar bloquejades de forma indefinida fins que el servidor es torni a estar disponible.

## Cas d'ús: Automount dels Homes dels usuaris del nostre sistema

1. Cada usuari del sistema tindrà el seu directori **home** compartit mitjançant **NFS**. Per fer-ho crearem una carpeta (homes) amb els permisos corresponents:

    ```sh
    mkdir /mnt/data/homes
    chmod 755 /mnt/data/homes
    ```

2. Assumirem que tenim 2 usuaris al sistema (david) i (ivan). Per tant, crearem 2 directoris al directori compartit:

    ```sh
    mkdir -d /mnt/data/homes/david /mnt/data/homes/ivan
    ```

3. Crearem el dos usuaris:

    ```sh
    useradd -m -d /home/david -s /bin/bash david
    useradd -m -d /home/ivan -s /bin/bash ivan
    ```

4. Edita el fitxer */etc/auto.master* per afegir la configuració d'automount:

    ```sh
    echo "/home /etc/auto.home" >> /etc/auto.master
    ```

5. Crea un fitxer /etc/auto.home amb el contingut següent:

    ```sh
    # Has de modificar ip-servidor
    echo "* -rw,soft ip-servidor:/mnt/data/homes/&" > /etc/auto.home
    ```

    * *: Indica que quan un usuari accedeix a una subcarpeta sota **/home**, aquesta clau és emparellada amb el nom d'usuari i actua com a referència per a la resta de l'expressió.
    * **-rw,soft**: Aquestes són les opcions de muntatge per als sistemes de fitxers que s'han de passar a mount quan es munta el recurs NFS.
        * **rw**: Permet tant la lectura com l'escriptura en el recurs NFS.
        * **soft**: En cas d'error (com una caiguda de la connexió NFS), es comporta de forma "suau" (soft), intentant accedir al recurs però retornant errors si no és possible. Aquesta opció permet que el sistema continuï funcionant malgrat possibles problemes amb el servidor NFS.
    * **ip-servidor:/mnt/data/homes/&**: És la ubicació del recurs NFS que es muntarà per a cada usuari.
        * **ip-servidor**: L'adreça IP del servidor NFS.
        * **/mnt/data/homes/**: La ruta de l'exportació NFS en el servidor.
        * **&**: Aquesta comodí representa el nom de l'usuari i es substituirà per a cada usuari.

6. Reinicia el servei *autofs*:

    ```sh
    systemctl restart autofs
    ```

### Proves

```sh
#!/bin/bash

# Llista d'usuaris
USUARIS=("david" "ivan")

# Simula l'accés als comptes dels usuaris
for USUARI in "${USUARIS[@]}"
do
    echo "Simulant l'accés de $USUARI..."
    su - $USUARI -c "ls -la ~"
done
```

## Reptes

1. Configureu el client per fer servir automount i autenticació centralitzada amb el servidor LDAP.
2. Prepareu amb Ansible scripts per automatitzar la creació del servidor i del client NFS.
3. Estableix un sistema de monitoratge per al teu servidor i client NFS, utilitzant [Prometheus](https://prometheus.io/).
4. Implementa una rèplica de servidors NFS per assegurar alta disponibilitat i tolerància a errors podeu consultar [Distributed Replicated Block Device](https://linbit.com/drbd/).
