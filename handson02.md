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