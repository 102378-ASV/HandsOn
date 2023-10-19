# Servidor de Backups

## Descripció

Configurarem un servidor de còpies de seguretat utilitzant rocky 8 i el programari bacula. [Bacula](https://www.bacula.org/) és un programari de còpies de seguretat de codi obert, que permet realitzar còpies de seguretat, restaurar-les i verificar-les. Aquesta eina permet realitzar còpies de seguretat de forma incremental, de manera que només es copien els canvis realitzats des de la darrera còpia de seguretat completa. Això permet reduir el temps de còpia de seguretat i el seu tamany. A més, permet tenir com a clients màquines amb sistemes operatius diferents, com ara Linux, Windows, macOS, Solaris, etc.

## Requisits previs

En aquest punt, assumirem que teniu una màquina virtual creada amb un sistema operatiu basat en Red Hat (com Rocky Linux) i que aquesta màquina té un disc virtual diferent del disc on està instal·lat el sistema operatiu. Podem utilitzar un altre disc 10GB que es muntarà a **/home**. Assumirem que en aquest servidor tenim activat *SELinux* i el firewall *firewalld*. 

En aquest servidor utilitzarem LVM i crearem un volum lògic per al sistema operatiu i un altre per a les dades. Això ens permetrà fer còpies de seguretat del sistema operatiu i de les dades per separat.

Ara mateix tenim el disc **/vda** amb la partició **vda1** que conté el sistema operatiu en un partició xfs. Aquest disc té un tamany de 4GB. A més, tenim un disc **/vdb** amb la partició **vdb1** que conté un sistema de fitxers xfs. Aquest disc té un tamany de 10GB. i un disc **/vdc** amb la partició **vdc1** que conté un sistema de fitxers xfs. Aquest disc té un tamany de 10GB.

Primerament crearem un volum lògic per al sistema operatiu  (**vgroot**) i un altre per a les dades (**vgdata**).

```sh
# Crearem un grup de volums per al sistema operatiu
vgcreate vgroot /dev/vdb

# Crearem un grup de volums per a les dades
vgcreate vgdata /dev/vdc

# Creem el volum lògic per al sistema operatiu
lvcreate -n lvroot -L 4G vgroot

# Creem el volum lògic per a les dades
lvcreate -n lvdata -L 10G vgdata
```

Ara migrarem el sistema operatiu que es troba a **/dev/vda1** al volum lògic **/dev/vgroot/lvroot**.

```sh
# Creem un sistema de fitxers xfs al volum lògic
mkfs.xfs /dev/vgroot/lvroot

# Muntarem el volum lògic a /mnt
mount /dev/vgroot/lvroot /mnt

# Desactivem el selinux pot fer que el rsync no funcioni correctament perquè no permetrà copiar alguns fitxers
setenforce 0

# Copiem el sistema operatiu al volum lògic on -a indica que es copiaran els atributs dels fitxers, -x indica que no es copiaran els sistemes de fitxers muntats, -H indica que no es copiaran els enllaços durs, -A indica que no es copiaran els fitxers de dispositiu i -X indica que no es copiaran els fitxers de sockets.

# Es pot donar que rsync no estigui instal·lat dnf install rsync -y  

rsync -avxHAX --exclude=/mnt / /mnt

# Actualitzem el fitxer fstab
# Eliminar la línia que fa referència a /dev/vda1
echo "/dev/mapper/vgroot-lvroot / xfs defaults 0 0" >> /mnt/etc/fstab

# Chroot al sistema operatiu
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
chroot /mnt

# Actualitzem initramfs
dracut -f 

# Actualitzem el grub
grub2-mkconfig -o /boot/grub2/grub.cfg

# Sortim del chroot
exit

# Reiniciem el sistema
reboot
```


## Instal·lació i Configuració del servidor Bacula

Per instal·lar el servidor bacula, executarem la següent comanda:

```sh
dnf install bacula-director bacula-storage bacula-console bacula-client -y
```

Ara configurarem el servidor bacula. Per això, editarem el fitxer **/etc/bacula/bacula-dir.conf** i afegirem el següent contingut:

```sh
Director {
  Name = rocky-dir
  DIRport = 9101
  QueryFile = "/etc/bacula/scripts/query.sql"
  WorkingDirectory = "/var/lib/bacula"
  PidDirectory = "/var/run"
  Maximum Concurrent Jobs = 20
  Password = "bacula-dir"
  Messages = Daemon
}

Director {
  Name = rocky-mon
  DIRport = 9102
  QueryFile = "/etc/bacula/scripts/query.sql"
  WorkingDirectory = "/var/lib/bacula"
  PidDirectory = "/var/run"
  Password = "bacula-mon"
  Messages = Daemon
}




