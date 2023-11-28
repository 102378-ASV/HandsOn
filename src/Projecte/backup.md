# Servidor de Backups

## Descripció

Configurarem un servidor de còpies de seguretat utilitzant Rocky 8 i el programari **bacula**. [Bacula](https://www.bacula.org/) és un programari de còpies de seguretat de codi obert, que permet realitzar còpies de seguretat, restaurar-les i verificar-les. Aquesta eina permet realitzar còpies de seguretat de forma incremental, de manera que només es copien els canvis realitzats des de la darrera còpia de seguretat completa. Això permet reduir el temps de còpia de seguretat i el seu tamany. A més, permet tenir com a clients màquines amb sistemes operatius diferents, com ara Linux, Windows, macOS, Solaris, etc.

## Requisits previs

En aquest punt, assumirem que teniu una màquina virtual creada amb un sistema operatiu basat en Red Hat (com Rocky Linux) i que aquesta màquina té un disc virtual diferent del disc on està instal·lat el sistema operatiu que utiltizarem per fer les còpies de seguretat. S'assumeix Rocky Linux 8 com a sistema operatiu.

En aquesta màquina virtual tenim el disc **/vda** amb la partició **vda1** que conté el sistema operatiu en un partició xfs. Aquest disc té un tamany de 4GB. A més, tenim un disc **/vdb** amb la partició **vdb1** que conté un sistema de fitxers xfs. Aquest disc té un tamany de 10GB. Aquest disc serà el disc on guardarem les còpies de seguretat.

Utilitzarem LVM per gestionar el disc **/vdb**. D'aquesta manera, podrem tenir un sistema de fitxers flexible i escalable.

- Instal·larem el paquet **lvm2** per gestionar el disc **/vdb**.
  
```sh
dnf install lvm2 -y
```

- Crearem un volum físic amb el disc **/vdb**.

```sh
pvcreate /dev/vdb
```

- Crearem un grup de volums anomenat **vgdata** amb el volum físic **/dev/vdb**.

```sh
vgcreate vgdata /dev/vdb
```

- Crearem un volum lògic anomenat **lvdata** amb el grup de volums **vgdata**. Aquest volum tindrà un tamany de 10GB.

```sh
lvcreate -n lvdata -L 10G vgdata
```

- Crearem un sistema de fitxers xfs al volum lògic **lvdata**.

```sh
mkfs.xfs /dev/vgdata/lvdata
```

- Muntarem el volum lògic **lvdata** a la carpeta **/mnt**.

```sh
mount /dev/vgdata/lvdata /mnt
```

## Instal·lació i Configuració del servidor Bacula

Un dels requeriments per a la instal·lació de bacula és tenir instal·lat el servidor de bases de dades MySQL. Per instal·lar-lo, executarem la següent comanda:

```sh
dnf install mariadb -y
```

Podem afegir a **/etc/my.cnf.d/mariadb-server.cnf** que indiqui que el servidor de bases de dades utilitzi el conjunt de caràcters **utf8mb4**. Això ens permetrà utilitzar caràcters especials en els noms de les bases de dades i en els noms dels fitxers de còpia de seguretat. Aquest pas es pot saltar si no es vol utilitzar caràcters especials en els noms de les bases de dades i en els noms dels fitxers de còpia de seguretat.

```cnf
[mysqld]
character-set-server = utf8mb4

[client]
default-character-set = utf8mb4
```

Un cop instal·lat, activarem el servei i comprovarem que està en execució:

```sh
systemctl enable --now mariadb
systemctl status mariadb
```

Ara ens connectarem al servidor de bases de dades i crearem la base de dades de bacula:

```sh
mysql -u root
```

```mysql
create database bacula; 
grant all privileges on bacula.* to bacula@'%' identified by 'Passw0rd';
flush privileges;
exit
```

Per instal·lar el servidor bacula, executarem la següent comanda:

```sh
dnf install bacula-director bacula-storage bacula-console bacula-client -y
```

Seleccionarem la base de dades que utilitzarà bacula. En aquest cas, seleccionarem **MySQL**.

```sh
alternatives --config libbaccats.so
```

Ara crearem les taules de la base de dades de bacula:

```sh
/usr/libexec/bacula/make_mysql_tables -u root
``````

Finalment, necessitem crear els directoris que utilitzarem per guardar les còpies de seguretat. En el nostre cas, hem creat un volum lògic anomenat **/dev/vgdata/lvdata** que muntarem a **/mnt**. Aquesta serà la carpeta on es guardaran les còpies de seguretat.

```sh
mkdir /mnt/bacula /mnt/bacula/backup /mnt/bacula/restore
```

Afegim els permissos per a que el servidor de bacula pugui accedir a aquesta carpeta:

```sh
chown -R bacula:bacula /mnt/bacula
chmod -R 700 /mnt/bacula
```

## Configurant el director de bacula

El director de bacula és l'encarregat de gestionar les còpies de seguretat.

Per configurar el director de bacula, editarem el fitxer **/etc/bacula/bacula-dir.conf** i afegirem el següent contingut:

```sh
cp /etc/bacula/bacula-dir.conf /etc/bacula/bacula-dir.default
```

- **Name**: Aquest és el nom del director. Pots canviar-ho segons les teves preferències.

- **DIRport**: Aquest paràmetre defineix el port en què escoltarà el director de Bacula. El valor per defecte és 9101, però pots canviar-lo si és necessari.

- **QueryFile**: Aquest és l'arxiu de consulta SQL que el director utilitzarà per recuperar informació del catàleg.

- **WorkingDirectory**: Aquesta és la carpeta de treball del director, on es guardarà la informació temporal. Assegura't que aquesta carpeta existeix i té els permisos adequats.

- **PidDirectory**: Aquesta carpeta conté els fitxers PID per al director. Assegura't que tingui els permisos adequats i que existeixi.

- **Maximum Concurrent Jobs**: Aquest paràmetre defineix el nombre màxim de treballs simultanis que el director pot gestionar. Pots ajustar-ho segons la capacitat del teu sistema.

- **Password**: Aquest és el contrasenya de la consola Bacula que s'utilitzarà per connectar-se al director. Has d'assegurar-te que aquesta contrasenya sigui segura.

- **Messages**: Aquesta opció determina on es registren els missatges del director. "Daemon" indica que els missatges s'enregistraran a un arxiu de registre.

- **DirAddress**: Aquest és l'adreça IP en la qual el director escoltarà les sol·licituds d'altres components de Bacula.

```json
Director {
  Name = bacula-dir  
  DIRport = 9101
  QueryFile = "/etc/bacula/scripts/query.sql"
  WorkingDirectory = "/var/lib/bacula"
  PidDirectory = "/var/run"
  Maximum Concurrent Jobs = 20
  Password = "@@DIR_PASSWORD@@" 
  Messages = Daemon
  DirAddress = 127.0.0.1
}
```

### Configurant catàleg de bacula

Un catàleg de bacula és una base de dades que conté informació sobre les còpies de seguretat que s'han realitzat. Aquesta informació inclou els fitxers que s'han copiat, els fitxers que s'han restaurat, els fitxers que s'han eliminat, etc. Aquesta informació es pot utilitzar per restaurar fitxers o per realitzar informes sobre les còpies de seguretat.

```json
Catalog {
  Name = MyCatalog
  dbname = "bacula"; dbuser = "bacula"; dbpassword = "Passw0rd"
}
```

Estem creant un catàleg anomenat **MyCatalog** que utilitzarà una base de dades anomenada **bacula**. Aquesta base de dades tindrà un usuari anomenat **bacula** amb la contrasenya **Passw0rd**. Aquest usuari tindrà permisos per crear taules i inserir dades a la base de dades.

### Configurant el dispositiu de còpia de seguretat

Autochanger ens permet configurar els dispositius de còpia de seguretat. Aquests dispositius poden ser cintes, discos, etc. En aquest cas, utilitzarem un disc per a les còpies de seguretat.

```json
Autochanger {
  Name = File1
  Address = 192.168.101.89               # IP del servidor
  SDPort = 9103
  Password = "@@SD_PASSWORD@@"
  Device = FileChgr1
  Media Type = File1
  Maximum Concurrent Jobs = 10        
  Autochanger = File1                 
}
```

Aquesta configuració defineix un dispositiu anomenat **File1** que utilitzarà el servidor amb la IP **192.168.101.89**. Aquest dispositiu utilitzarà el port **9103** i la contrasenya **@@SD_PASSWORD@@**. Aquest dispositiu utilitzarà el dispositiu **FileChgr1** i el tipus de mitjà **File1**. Aquest dispositiu pot gestionar fins a 10 treballs simultanis.

### Configurant el storage de bacula

```sh
cp /etc/bacula/bacula-sd.conf /etc/bacula/bacula-sd.default
```

En primer lloc, defirinrem el nom del storage. Com que el dispositiu que hem configurat al director és **FileChgr1**:


```json
Autochanger {
  Name = FileChgr1
  Device =FileChgr1-Dev1
  Changer Command = ""
  Changer Device = /dev/null
}
```

Com volem que el dispostiu sigui el disc que hem muntat a /mnt i que les còpies de seguretat es guardin a /mnt/bacula/backup, afegirem el següent contingut:

```json
Device {
  Name = FileChgr1-Dev1
  Media Type = File1
  Archive Device = /mnt/bacula/backup
  LabelMedia = yes;                   
  Random Access = Yes;
  AutomaticMount = yes;               
  RemovableMedia = no;
  AlwaysOpen = no;
  Maximum Concurrent Jobs = 5
}
```

Aquesta configuració defineix un dispositiu anomenat **FileChgr1-Dev1** que utilitzarà el tipus de mitjà **File1**. Aquest dispositiu guardarà les còpies de seguretat a **/mnt/bacula/backup**. Aquest dispositiu pot gestionar fins a 5 treballs simultanis.

## Configurant un client Bacula per a fer còpies de seguretat

### Simulant un escenari en una nova màquina virtual (client)

Imagineu que tenim un servidor amb rocky on accedeixen múltiples usuaris i ens interessa tenir còpies de seguretat del seus *home*. Per això connectarem amb el nostre servidor (director) de bacula i crearem un client per a fer còpies de seguretat dels *home* dels usuaris.

En primer lloc, crearem alguns usuaris:

```sh
useradd -m user1 #passwd user1 1234
useradd -m user2 #passwd user2 1234
```

Ara crearem un fitxer amb dades per a cada usuari:

```sh
su - user1 -c "echo \"This is user1's data\" > ~/user1_data.txt"
su - user2 -c "echo \"This is user2's data\" > ~/user2_data.txt"
```

Ara instal·larem el client bacula:

```sh
dnf install bacula-client -y
```

En aquest punt assumirem que desactivem el firewall i el selinux. Per facilitar la configuració inicial. Però, recorda que hauries d'activar-los i configurar-los correctament. Un cop t'asseguris que tots els components de bacula funcionen correctament.

```sh
setenforce 0
systemctl stop firewalld
```

Ara guardarem una còpia del fitxer de configuració per defecte:

```sh
cp /etc/bacula/bacula-fd.conf /etc/bacula/bacula-fd.default
```

Ara editarem el fitxer **/etc/bacula/bacula-fd.conf** i afegirem el següent contingut:

```json
Director {
  Name = bacula-dir
  Password = "AsedefSAsddsae-X89afde_asdE-h" 
}
```

on:

- **Name**: Aquest és el nom del director. Pots canviar-ho segons les teves preferències. Però ha de coincidir amb el nom que hem definit al director (servidor).
- **Password**: Aquesta és la contrasenya del director. Has d'assegurar-te que aquesta contrasenya sigui segura i la mateixa que has definit al director (Secció Client).


```json
FileDaemon {                          
  Name = computing-fd
  FDport = 9102                  
  WorkingDirectory = /var/spool/bacula
  Pid Directory = /var/run
  Maximum Concurrent Jobs = 20
  Plugin Directory = /usr/lib64/bacula
  FDAddress = 192.168.101.106
}
```

on:

- **Name**: Aquest és el nom del client. Pots canviar-ho segons les teves preferències.
- **FDport**: Aquest paràmetre defineix el port en què escoltarà el client de Bacula. El valor per defecte és 9102, però pots canviar-lo si és necessari.
- **WorkingDirectory**: Aquesta és la carpeta de treball del client, on es guardarà la informació temporal. Assegura't que aquesta carpeta existeix i té els permisos adequats.
- **PidDirectory**: Aquesta carpeta conté els fitxers PID per al client. Assegura't que tingui els permisos adequats i que existeixi.
- **Maximum Concurrent Jobs**: Aquest paràmetre defineix el nombre màxim de treballs simultanis que el client pot gestionar. Pots ajustar-ho segons la capacitat del teu sistema.
- **Plugin Directory**: Aquesta és la carpeta on es troben els plugins del client.
- **FDAddress**: Aquest és l'adreça IP en la qual el client escoltarà les sol·licituds del director de Bacula. (**IP del client**)

Un cop configurat el client, reiniciarem el servei:

```sh
systemctl restart bacula-fd
```

### Configurant bacula per a fer còpies dels /home del client 

Ara retornarem al nostre servidor (director) de bacula i configurarem el director per a fer còpies de seguretat dels /home del client.

Per configurar el director de bacula, editarem el fitxer **/etc/bacula/bacula-dir.conf** i afegirem el següent contingut:

1. Configurarem el client:

    ```json
    Client {
      Name = computing-fd
      Address = 192.168.101.106
      FDPort = 9102
      Catalog = MyCatalog
      Password = "AsedefSAsddsae-X89afde_asdE-h"
      File Retention = 30 days
      Job Retention = 6 months
      AutoPrune = yes
    }
    ```

    on:

    - **Name**: Fer-lo coincidir amb el nom definit a la màquina client.
    - **Address**: Fer-lo coincidir amb la IP de la màquina client.
    - **Password**: Fer-lo coincidir amb la contrasenya definida a la màquina client.

2. Crearem un job anomenat **BackupUserHomes** que farà còpies de seguretat dels /home del client:

    ```json
    Job {
      Name = "BackupUserHomes"
      Type = Backup
      Client = computing-fd
      FileSet = "UserHomesBackup"
      Schedule = "WeeklyCycle"  
      Storage = File1
      Messages = Standard
      Pool = File
      SpoolAttributes = yes
      Priority = 10
      Write Bootstrap = "/var/spool/bacula/RemoteBackup.bsr"
      Where = /mnt/bacula/restore
    }
    ```

    on:

    - **Schedule**: Aquest paràmetre defineix cada quan s'executarà el job. En el nostre cas, el job s'executarà cada setmana.
    - **Storage**: Aquest paràmetre defineix el dispositiu de còpia de seguretat que s'utilitzarà per guardar les còpies de seguretat. En el nostre cas, utilitzarem el dispositiu **File1**.
    - **Where**: Aquest paràmetre defineix la carpeta on es restauraran les còpies de seguretat. En el nostre cas, utilitzarem la carpeta **/mnt/bacula/restore**.

3. Definirem el FileSet anomenat **UserHomesBackup** que farà còpies de seguretat dels /home del client:

    ```json
    FileSet {
      Name = "UserHomesBackup"
      Include {
        Options {
          signature = MD5
        }
        File = /home  # Path to user home directories on the remote machine
      }
      Exclude {
        File = /home/tmp
        File = /home/cache
      }
    }
    ```

4. Per finalitzar arancarem el servei:

    ```sh
    systemctl restart bacula-dir
    ```

## Realitzant còpies de seguretat

Per realitzar còpies de seguretat, utilitzarem la consola de bacula:

1. Per accedir a la consola de bacula, executarem la següent comanda:

    ```sh
    bconsole
    ```

2.Per veure tots els clients que tenim configurats, executarem la següent comanda:

  ```sh
  *list clients
  ```

3. Per comprovar que el client està connectat, executarem la següent comanda:

    ```sh
    *status client=computing-fd
    ```

4. Per llançar el job de còpia de seguretat, executarem la següent comanda:

    ```sh
    *run job=BackupUserHomes
    ```

5. Per veure l'estat del job, executarem la següent comanda:

    ```sh
    *status dir
    ```

6. Per sortir de la consola de bacula, executarem la següent comanda:

    ```sh
    *quit
    ```
