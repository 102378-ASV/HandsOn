# Eines de xifratge

## Xifrant discos i particions

El xifratge de discos protegeix les dades d'un dispositiu. Per accedir al contingut desxifrat del dispositiu, l'usuari ha de proporcionar una frase de pas o una clau d'autenticació. El xifratge per defecte utilitzat per LUKS és aes-XTS-plain64. La mida de la clau per defecte per a LUKS és de 512 bits.

### Punts forts

* **Protecció Complerta**: LUKS encripta tot el dispositiu, proporcionant una protecció robusta per a les dades, especialment en dispositius mòbils i portàtils.
* **Flexibilitat d'Ús**: És útil per a xifrar dispositius d'intercanvi i altres usos especialitzats com arxius de bases de dades amb necessitats específiques de format. **Nota**: 
* **Gestió de Claus**: LUKS permet gestionar múltiples claus o frases de contrasenya per als usuaris, afegint així una capa addicional de seguretat.
* **Compatibilitat**: LUKS és compatible amb altres eines de xifratge de discos, com ara TrueCrypt i BitLocker.
* LUKS utilitza el subsistema de el nucli de mapeig de dispositius existent.
* LUKS proporciona un reforç de la contrasenya que protegeix contra els atacs de diccionari.
* Els dispositius LUKS contenen diverses ranures per a claus, el que permet als usuaris afegir claus de seguretat o frases de contrasenya,

### Punts febles

* **Desxifrat en Ús**: LUKS només protegeix les dades quan el sistema està apagat. Quan el sistema està en marxa, els arxius d'aquest disc estan disponibles per a qualsevol usuari.
* **Gestió d'Usuaris**: No és adequat per a escenaris que requereixen que molts usuaris tinguin claus d'accés diferents per al mateix dispositiu.
* **Encriptació a Nivell d'Arxiu**: No és adequat per a aplicacions que requereixen encriptació a nivell d'arxiu.

### Cas d'ús xifrant un disc amb LUKS

Per fer aquesta activitat assegureu-vos que teniu una màquina virtual amb un disc muntat diferent del sistema de fitxers root. En aquest exemple, utilitzarem un disc de 0,5GB muntat a /mnt/private. Aquest disc es troba en un dispositiu virtual anomenat /dev/vdb.

**Nota**: Si aprofiteu el mateix disc que heu fet anar en l'activitat anterior [Sistema de fitxers](./sistema_fitxers.md) heu de desmuntar-lo i formatejar-lo abans de continuar. 

```sh
umount /home
mkfs.xfs -f /dev/vdb
```

1. **Instal·leu el paquet cryptsetup**. Aquest paquet conté les eines necessàries per a la gestió de dispositius LUKS.

    ```sh
    dnf install cryptsetup -y
    ```

2. Formateu el disc amb cryptsetup i introduïm una frase de contrasenya segura (aquesta és la contrasenya que haureu d’introduir per desbloquejar el disc).

    ```sh
    cryptsetup luksFormat /dev/vdb
    ```

    ```shell
    WARNING!
    ========
    This will overwrite data on /dev/vdb irrevocably.

    Are you sure? (Type uppercase yes): YES
    Enter passphrase for /dev/vdb:
    Verify passphrase:

    # Si poseu coses simples no us deixarà -> per exemple una bona passphrase 
    # seria asv2324 i la compliquem una mica -sV2324i
    ```

3. Obrim el disc xifrat amb cryptsetup i introduïm el nom de mapper del dispositiu i del dispositiu (aquest és el */dev/mapper/* nom que voleu que tingui el vostre disc) i introduïu la contrasenya que del pas anterior.

    ```sh
    cryptsetup luksOpen /dev/vdb privateDisk 
    ```

4. Crearem un sistema de fitxer a la nostra partició xifrada

    ```sh
    mkfs.xfs /dev/mapper/privateDisk
    ```

    ```shell
        meta-data=/dev/mapper/privateDisk isize=512    agcount=4, agsize=30976 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
        data     =                       bsize=4096   blocks=123904, imaxpct=25
                =                       sunit=0      swidth=0 blks
        naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
        log      =internal log           bsize=4096   blocks=1368, version=2
                =                       sectsz=512   sunit=0 blks, lazy-count=1
        realtime =none                   extsz=4096   blocks=0, rtextents=0
    ````

5. Crearem el nou directori utilitzat per al punt de muntatge del sistema de fitxers xifrat.  

    ```sh
    mkdir -p /mnt/private
    ````

6. Ara monteu el disc xifrat al directori de muntatge que heu creat.  

    ```sh
    mount /dev/mapper/privateDisk /mnt/private/
    ```

7. Comproveu que el disc s'ha muntat correctament.  

    ```sh
    df -h
    ```

    ```shell
    Filesystem               Size  Used Avail Use% Mounted on
    devtmpfs                 469M     0  469M   0% /dev
    tmpfs                    485M     0  485M   0% /dev/shm
    tmpfs                    485M   13M  473M   3% /run
    tmpfs                    485M     0  485M   0% /sys/fs/cgroup
    /dev/vda1                4.0G  1.2G  2.9G  30% /
    tmpfs                     97M     0   97M   0% /run/user/0
    /dev/mapper/privateDisk  479M   28M  451M   6% /mnt/private
    ```

En aquest punt ens podriam plantegar si volem que el disc es munti automàticament al arrencar el sistema. Si és així, heu de crear una entrada a **/etc/fstab** i **/etc/crypttab**.

1. **/etc/fstab**: Aquest fitxer conté una llista de dispositius i els punts de muntatge associats. Aquest fitxer s'utilitza per muntar automàticament els dispositius al arrencar el sistema.  

    ```sh
    echo "/dev/mapper/privateDisk /mnt/private xfs defaults 0 2" >> /etc/fstab
    ```

    **Nota**: Com la partició xifrada contrindrà dades, podem posar el valor a 2 enlloc de a 0 per indicar que no es una partició prioritaria que calgui comprovar al arrencar el sistema.

2. **/etc/crypttab**: Aquest fitxer conté una llista de dispositius i els punts de muntatge associats. Aquest fitxer s'utilitza per muntar automàticament els dispositius al arrencar el sistema.  

    ```sh
    echo "privateDisk /dev/vdb none luks" >> /etc/crypttab
    ```

    **Nota**: *none* significa que no s'està fent servir cap fitxer de clau o contrasenya automàtica per desxifrar la partició, i la contrasenya o clau haurà de ser introduïda manualment quan es munti la partició.

3. Reinicieu el sistema per comprovar que el disc s'ha muntat automàticament.  

    ```sh
    reboot
    ```

4. Al arrancar obriu la terminal VNC d'OpenNebula i haureu d'introudir la contrasenya que heu posat al pas 2. Un cop fet el sistema arrancarà.

LUKS permet utilitzar fitxers clau per desxifrar els vostres dispositius, i és important investigar com fer-ho de manera segura. Tanmateix, cal tenir present que, tot i ser una eina útil per desxifrar dispositius sense necessitat d'introduir manualment una contrasenya, aquesta contrasenya es troba continguda en un fitxer. Això implica que si una persona no autoritzada aconsegueix accedir a aquest fitxer, també podrà accedir al contingut del disc protegit. Per aquest motiu, l'ús de fitxers clau no és una opció molt recomanada per a la majoria dels casos, ja que pot representar un risc per a la seguretat de les dades. Malgrat això, en situacions concretes i amb les degudes precaucions, l'ús de fitxers clau pot resultar útil. És fonamental avaluar amb deteniment el context i les necessitats de seguretat abans de decidir implementar aquesta opció.

## Reptes

* Investigeu i documenteu com afegir un fitxer clau per desxifrar el disc i que es faci l'automount sense demanar la contrasenya.
