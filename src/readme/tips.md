# Consells

## Consell 1

Podeu millorar la gestió dels servidors amb la comanda ```hostnamectl set-hostname nom_servidor``` , on *nom_servidor* hauria de ser un nom descriptiu com ara **DB_SERVER** si és el servidor de la base de dades. Això ens permet mantenir la claredat i l'ordre en entorns amb moltes terminals i servidors diferents, assegurant que sabem en tot moment a quin servidor estem treballant.

## Consell 2

És essencial ser eficients en la gestió. Si la **màquina virtual** amb *Ansible* ha de servir per automatitzar totes les màquines que crearem, no és qüestió de  copiar i enganxar la clau pública a totes les noves màquines (com vam fer a classe). Cal ser astuts. Cal afegir la clau pública de la màquina Ansible a **auth** de l'OpenNebula, juntament amb la clau del nostre pc i així totes les futures màquines que crearem ansible s'hi podrà connectar.

## Consell 3

Podeu fer servir eines com ```screen``` o ```nohup``` per executar comandes que requereixen temps i continuar l'execució sense dependre de la sessió actual.

```sh
nohup ./comanda_que_requereix_temps.sh &
```

o bé

```sh
screen -S nom_sessió
# Observeu que la terminal ha canviar
# Executeu la comanda que requereix temps
./comanda_que_requereix_temps.sh

# Crtl+alt+d surtireu de la sessió, però la sessió seguirà executant
# Per torna a la sessió
screen -r nom_sessió
```

## Consell 4

És crucial assegurar-vos que, cada vegada que actualitzeu la taula de particions o els sistemes de fitxers associats, comproveu que l'UUID de la partició no hagi canviat. En cas de canvi, serà necessari actualitzar l'arxiu ```/etc/fstab``` amb el nou UUID corresponent. També us recomanem revisar l'arxiu ```/etc/fstab``` si heu eliminat o creat noves particions abans de reiniciar el sistema. No fer-ho pot provocar problemes d'arrencada. Per verificar que tots els sistemes de fitxers es muntin correctament abans de reiniciar, podeu utilitzar la comanda ```mount -a```.
