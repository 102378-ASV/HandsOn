# MFA

L'autenticació de múltiples factors (MFA) és un mètode d'autenticació que requereix que l'usuari proporcioni dos o més factors d'autenticació per obtenir l'accés. Aquests factors poden ser alguna cosa que l'usuari sap (contrasenya), alguna cosa que l'usuari té (un codi temporal generat per una aplicació) o alguna cosa que l'usuari és (una empremta digital).

* **Increment de la Seguretat**: La MFA proporciona una capa addicional de seguretat més enllà de la contrasenya. Incloure un segon factor, com un codi temporal generat per una aplicació, fa que sigui molt més difícil per a atacants malintencionats obtenir l'accés no autoritzat.

* **Prevenen Atacs de Força Bruta**: Incloure la MFA ajuda a mitigar els atacs de força bruta, ja que fins i tot si un atacant adivina o roba una contrasenya, necessitarà també el segon factor per accedir.

* **Protecció contra Phishing**: La MFA amb un token generat localment mitiga l'impacte del phishing, ja que fins i tot si un atacant obté la contrasenya, necessitarà també el segon factor que només es troba a l'aplicació d'autenticació de l'usuari.

## Compte

* No tanqueu mai totes les sessions ssh de root, fins asseguar-vos que les modificacions que heu fet a sshd_config són correctes. Si no, podreu quedaros fora del servidor.
* Es requereix l'app de google autenticator instal·lada al vostre dispositiu mòvil.
  


## Configurant (password + token) per a SSH amb el módul Google Authenticator

1. Instal·la el paquet `google-authenticator`:

    ```bash
    dnf install google-authenticator -y
    ```

2. Instal·la el paquet `qrencode`, aquest paquet ens permetrà generar codis QR:

    ```bash
    dnf install qrencode qrencode-libs -y
    ```

3. Crearem un usuari nou i li assignarem una contrasenya:

    ```bash
    useradd -m sshuser
    passwd sshuser
    su - sshuser
    ```

4. Executa la comanda `google-authenticator` per configurar l'autenticació de múltiples factors:

    ```bash
    google-authenticator
    ```

    * **Do you want authentication tokens to be time-based (y/n)** y. Això farà que els tokens generats per l'aplicació de Google Authenticator caduquin cada 30 segons.
  
    Si teniu una terminal petita, o no heu instal·lat el paquet de QR no us deixarà veure el codi QR, però podeu copiar el codi que us dona i afegir-lo a l'aplicació de Google Authenticator.

    Un cop afegit a la vostra aplicació de Google Authenticator, us generarà un codi de 6 dígits que haureu d'introduir a la terminal per completar l'associació.

    * **Do you want me to update your "/home/sshuser/.google_authenticator" file? (y/n)** y. Això actualitzarà el fitxer `.google_authenticator` de l'usuari sshuser amb la configuració que heu seleccionat.

    * **Do you want to disallow multiple uses of the same authentication token? (y/n)** y. Això farà que cada token només pugui ser utilitzat una vegada.

    * **By default, a new token is generated every 30 seconds by the mobile app.In order to compensate for possible time-skew between the client and the server,we allow an extra token before and after the current time. This allows for a time skew of up to 30 seconds between authentication server and client. If you experience problems with poor time synchronization, you can increase the window from its default size of 3 permitted codes (one previous code, the currentcode, the next code) to 17 permitted codes (the 8 previous codes, the current
    code, and the 8 next codes). This will permit for a time skew of up to 4 minutes
    between client and server.Do you want to do so? (y/n) y. Això permetrà que el codi generat per l'aplicació de Google Authenticator sigui vàlid durant 4 minuts.

    * **If the computer that you are logging into isn't hardened against brute-force login attempts, you can enable rate-limiting for the authentication module. By default, this limits attackers to no more than 3 login attempts every 30s. Do you want to enable rate-limiting? (y/n)** y. Això limitarà els atacants a no més de 3 intents de connexió cada 30 segons.

5. Retorna a l'usuari root:
   
    ```bash
    exit
    ```

6. Configuració del servei SSH per utilitzar l'autenticació de múltiples factors. Per fer-ho, edita el fitxer `/etc/ssh/sshd_config` i canvia la línia `ChallengeResponseAuthentication` perquè quedi així:

    ```bash
    ChallengeResponseAuthentication yes
    ```

7. En el mateix fitxer `/etc/ssh/sshd_config` canvia la línia `PasswordAuthentication` perquè quedi així:

    ```bash
    PasswordAuthentication yes
    ```

8. Finalment, editarem el fitxer de connfiguració per indicar que volem utilitzar autenticació per clau pública a l'usuari root i de doble factor (contrasenya + token) pels usuaris del grup (sshuser):

    ```bash
    Match User root
    AuthenticationMethods publickey
    Match Group sshuser
    AuthenticationMethods password keyboard-interactive
    ```

9. Reinicia el servei SSH per aplicar els canvis:

    ```bash
    systemctl restart sshd
    ```

    En aquest punt si feu un test, observareu que l'usuari **root** es pot seguir connectant amb clau pública al servidor ```ssh root@192.168.101.77 -i .ssh/jmf-stormy``` i que l'usuari **sshuser** pot connectar ```ssh sshuser@192.168.101.77```. Cal observar que us demanarà la contrasenya i **NO** el token. Per tant, hem de configurar el PAM per utilitzar l'autenticació de múltiples factors.

10. Actualitzarem el fitxer `/etc/pam.d/sshd` perquè utilitzi el mòdul `pam_google_authenticator.so`:

    ```bash
    vi /etc/pam.d/sshd
    ```

    Afegim al final: `auth required pam_google_authenticator.so nullok`. Aquesta línia indica que el mòdul `pam_google_authenticator.so` és obligatori per a l'autenticació, però que si no està configurat, no es bloquejarà l'accés.

    Tancar i guardar el fitxer.

11. Reinicia el servei SSH per aplicar els canvis:

    ```bash
    systemctl restart sshd
    ```

12. Ara observem que encara ens falla la conexió i l'usuari **sshuser** no pot accedir al servidor. Si desactiveu SELinx ```setenforce 0``` veureu que l'usuari **sshuser** pot accedir al servidor. Per tant, cal configurar SELinux perquè permeti l'accés. El problema rau en SELinux prohibint al servei **sshd** accedir al fitxer `.google_authenticator` de l'usuari sshuser. Bàsicament **sshd** no pot accedir a fitxers que es trobin ubicats fòra de *.ssh*. Per solucionar-ho:

    ```bash
    su - sshuser
    mkdir .ssh
    mv .google_authenticator .ssh/
    exit
    ```

13. Editem  `/etc/pam.d/sshd` modificant la línia anterior per `auth required pam_google_authenticator.so nullok secret=/home/${USER}/.ssh/.google_authenticator`.

14. Reiniciem el servei SSH per aplicar els canvis:

    ```bash
    systemctl restart sshd
    ```

## Reptes

* Cerca un altra alternativa per solucionar el problema de SELinux.
* Investiga i descriu com s'implementa l'autenticació MFA amb altres aplicacions diferents de Google Authenticator.
* Automatitza la configuració de l'autenticació MFA per a SSH.