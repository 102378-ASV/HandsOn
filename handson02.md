# HANDSON 02
## Repte (Oh my Zsh i Powerlevel10k): 

Per instal·lar el Oh my Zsh, he instal·lat el zsh (i git) i després he executat la comanda que ens donen a la pàgina web del Oh my Zsh.:
dnf install zsh -y

i després:
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

Per instal·lar el Powerlevel10k, he executat la comanda que ens donen a la pàgina web del Powerlevel10k:
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-~/.oh-myzsh-custom}/themes/powerlevel10k

i després he editat el fitxer .zshrc i he canviat el tema a powerlevel10k:
ZSH_THEME="powerlevel10k/powerlevel10k"

 ## Repte (configurar Client LAM):

1. Instal·lar paquets necessaris: wget, cyprus-sasl.devel, libtool-ltdl-devel, openssl-devel, libdb-devel, make, libtool, autoconf, tar, gcc, perl i perl-devel.
2. Descarregar el paquet de LAM: wget https://www.openldap.org/software/download/OpenLDAP/openldap-release/openldap-$VER.tgz (on $VER és la versió de LAM que volem instal·lar (hem declarat la 2.6.6)).
3. Descomprimir el paquet: tar -xvf openldap-$VER.tgz
4. Entrar al directori: cd openldap-$VER
5. Instal·lar OpenLDAP
6. Configurar OpenLDAP
7. Definir l'esquema de LAM
8. Configurar l'usuari
