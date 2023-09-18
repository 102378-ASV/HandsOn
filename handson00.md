# HANDSON 00: Repte (Disseny 2): 

Per tal d'assolir el repte, hem exportat la base de dades que ja teniem amb la comanda mysqldump -u root -p wordpress_db > backup.sql.
Per fer la nova màquina virtual, hem fet un resize de la VM que ja teniem, assignant ara 3 GB de RAM i 0,75 de CPU.
Preparem la nova VM, instal.lant el firewalld i mariadb-server.
Transferim el backup.sql, pujant a un servidor l'arxiu fent servir lftp.
Importem la base de dades a la nova VM.
Reconfigurem el wp-config.php (hem tingut problemes i no carrega la base de dades a la pàgina web, tot i havent reconfigurat el wp-config ens surt un error).

La pràctica ha estat feta per: David Argente i Eric López.