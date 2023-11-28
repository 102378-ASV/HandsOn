# Servidor Web (jupyter)

En aquest apartat es descriu la configuració d'un servidor web que permeti executar codi python amb jupyterhub. Aquesta instancia permetrà executar codi python i tenir un entorn de treball per a desenvolupar programes. També permetrem als usuari accés per SSH i SFTP als seus directoris de treballs, per si volen utilitzar un IDE per a desenvolupar els seus programes.

## Introducció

Configurar un servidor web amb JupyterHub ofereix un entorn de treball flexible i potent per als desenvolupadors de Python. Aquesta guia detalla els passos necessaris per establir un entorn que permeti als usuaris executar codi Python, accedir per SSH i SFTP als seus directoris de treball, i desenvolupar programes amb facilitat. Utilitzarem Rocky Linux 8 com a sistema operatiu.

## Objectius

- Instal·lar i configurar un servidor web basat en JupyterHub.
- Configurar el sistema de fitxers per a JupyterHub.
- Configurar JupyterHub de forma segura.
- Confgiuració de seguretat del servidor.
- Desplegament d'un servei web basat en un servei de linux.

## Preparatius inicials

1. Crear una màquina virtual amb *Rocky Linux 8* amb el nom `jupyter-web` i amb 1GB de RAM, 0.25 CPU i 1 VCPU.

2. Assignar-li una segona unitat de disc de 5GB.

3. Configurar el nom de la màquina virtual:

    ```bash
    hostnamectl set-hostname jupyter.tc.udl.cat
    ```

4. Actualitzarem el sistema:

    ```bash
    dnf update -y
    ```

## Configuració del sistema de fitxers

Una bona pràctica per configurar el sistema de fitxers és utilitzar LVM. Això permetrà que el sistema de fitxers sigui flexible i pugui créixer en cas de necessitat. Per això, crearem un volum per a JupyterHub i un volum per a cada usuari.

1. Instal·larem el paquet `lvm2`:

    ```bash
    dnf install lvm2 -y
    ```

2. Crearem un volum per a JupyterHub. Qualsevol servei a linux necessitarà guardar els executables, les llibreries, els logs i informació sobre la seva execució. Us proposo definir 4 volums lògics. El volum jupyter-service contindrà els executables i les llibreries del servei. El volum jupyter-venv contindrà els entorns virtuals de python. El volum jupyter-run contindrà els fitxers temporals i el volum jupyter-logs contindrà els logs del servei.

    ```bash
    vgcreate jupyterhub /dev/vdb
    lvcreate -L 500M -n jupyter-service jupyterhub
    lvcreate -L 500M -n jupyter-venv jupyterhub
    lvcreate -L 500M -n jupyter-run jupyterhub
    lvcreate -L 1G -n jupyter-logs jupyterhub
    ```

3. Crearem els sistemes de fitxers per a cada volum.

    ```bash
    mkfs.xfs /dev/jupyterhub/jupyter-service
    mkfs.xfs /dev/jupyterhub/jupyter-venv
    mkfs.xfs /dev/jupyterhub/jupyter-run
    mkfs.xfs /dev/jupyterhub/jupyter-logs
    ```

4. Crearem els directoris on muntarem els volums.

    ```bash
    mkdir /opt/jupyterhub
    mkdir /opt/jupyterhub/service
    mkdir /opt/jupyterhub/venv
    mkdir /opt/jupyterhub/run
    mkdir /opt/jupyterhub/logs
    ```

5. Crearem un usuari per a executar el servei.

    ```bash
    useradd jupyterhub -s /sbin/nologin
    ```

6. Muntarem els volums.

    ```bash
    echo "/dev/jupyterhub/jupyter-service /opt/jupyterhub/service xfs defaults 0 0" >> /etc/fstab
    echo "/dev/jupyterhub/jupyter-venv /opt/jupyterhub/venv xfs defaults 0 0" >> /etc/fstab
    echo "/dev/jupyterhub/jupyter-run /opt/jupyterhub/run xfs defaults 0 0" >> /etc/fstab
    echo "/dev/jupyterhub/jupyter-logs /opt/jupyterhub/logs xfs defaults 0 0" >> /etc/fstab
    mount -a
    ```

## Configuració de JupyterHub

1. Instal·larem el paquet `python3`:

    ```bash
    dnf install python3 -y
    ```

2. Instal·larem el paquet `python3-pip`:

    ```bash
    dnf install python3-pip -y
    ```

3. Instal·larem el paquet `nodejs`:

    ```bash
    dnf install nodejs -y
    ```

4. Instal·larem el paquet `npm`:

    ```bash
    dnf install npm -y
    ```

5. Instal·larem el paquet `configurable-http-proxy`:

    ```bash
    npm install -g configurable-http-proxy
    ```

6. Crearem un entorn virtual per a JupyterHub:

    ```bash
    cd /opt/jupyterhub
    python3 -m venv venv
    ```

7. Activarem l'entorn virtual:

    ```bash
    source /opt/jupyterhub/venv/bin/activate
    ```

8. Actualitzarem el paquet `pip`:

    ```bash
    pip3 install --upgrade pip
    ```

9. Instal·larem el paquet `jupyterhub`:

    ```bash
    pip3 install jupyterhub
    ```

    **Observació**: L'usuari *jupyterhub* és un usuari dedicat que assumeix la responsabilitat del servei sense disposar de permisos de **root**. Els usuaris que accedeixen al servei podrien necessitar privilegis de root, com ara per autenticar-se mitjançant els serveis PAM. La integració de **Sudospawner** amb **JupyterHub** permet elevació de permissos, garantint així una execució segura. A més, la configuració de JupyterHub per fer servir Sudospawner com a spawner predeterminat, seguida de la reinicialització del servei, assegura que els nous servidors d'usuaris es creïn amb els privilegis necessaris.

10. Instal·larem el paquet `sudospawner`:

    ```bash
    pip3 install sudospawner
    ```

11. Assignarem de propietari a l'usuari *jupyterhub* per utilitza tots els serveis relacionats amb JupyterHub:

    ```bash
    chown -R jupyterhub:jupyterhub /opt/jupyterhub
    ```

12. Activarem l'entorn virtual a l'inici de sessió de l'usuari *jupyterhub*:

    ```bash
    echo "source /opt/jupyterhub/venv/bin/activate" >> /home/jupyterhub/.bashrc
    ```

13. Utilitzarem **visudo** per a editar el fitxer */etc/sudoers* i afegir la següent línia:

    ```bash
    Cmnd_Alias JUPYTER_CMD = /opt/jupyterhub/venv/bin/sudospawner
    jupyterhub ALL=(%jupyterhub) NOPASSWD:JUPYTER_CMD
    ```

    D'aquesta manera, l'usuari *jupyterhub* podrà executar el programa *sudospawner* sense necessitat de contrasenya. Tots els usuaris hauran de ser membres del grup *jupyterhub*. Per tant, cal editar el nostre script de creació d'usuaris i afegir la següent línia:

    ```bash
    usermod -a -G jupyterhub $user
    ```


14. L'ús del grup "shadow" és crucial perquè JupyterHub pugui accedir a la informació de contrasenya dels usuaris i autenticar-los correctament quan s'accedeix al servei. Així, aquest conjunt de comandes contribueix a una integració segura i efectiva entre PAM, JupyterHub i el sistema d'autenticació del sistema operatiu.

    ```bash
    groupadd shadow
    chgrp shadow /etc/shadow
    chmod g+r /etc/shadow
    usermod -a -G shadow jupyterhub
    ```

15. Finalment, cal configurar el servei Selinux per a que permeti l'execució del programa *sudospawner*:

    ```bash
    dnf install policycoreutils-python-utils -y
    ```

    ```bash
    cat << EOF > sudo_exec_selinux.te
    module sudo_exec_selinux 1.0;
    require {
        type unconfined_t;
        type sudo_exec_t;
        class file { read entrypoint };
    }

    #============= unconfined_t ==============
    allow unconfined_t sudo_exec_t:file entrypoint;
    EOF
    ```

    ```bash
    checkmodule -M -m -o sudo_exec_selinux.mod sudo_exec_selinux.te
    semodule_package -o sudo_exec_selinux.pp -m sudo_exec_selinux.mod
    semodule -i sudo_exec_selinux.pp
    ```

## Configuració d'un servei systemd

1. Crearem un fitxer de configuració per al servei JupyterHub:

    ```bash
    cat << EOF > /etc/systemd/system/jupyterhub.service
    [Unit]
    Description=JupyterHub
    After=syslog.target network.target

    [Service]
    WorkingDirectory=/opt/jupyterhub
    User=jupyterhub
    Group=jupyterhub
    Environment="PATH=/opt/jupyterhub/venv/bin:/usr/local/bin/:/usr/bin"
    ExecStart=/bin/bash -c "/opt/jupyterhub/venv/bin/jupyterhub --JupyterHub.spawner_class=sudospawner.SudoSpawner"

    [Install]
    WantedBy=multi-user.target
    EOF
    ```

    A /usr/bin hi ha el programa nodejs. A /usr/local/bin hi ha el programa configurable-http-proxy. A /opt/jupyterhub/venv/bin hi ha el programa sudospawner. Així, el PATH del servei JupyterHub ha de contenir aquests directoris.

2. Recarregarem la configuració dels serveis:

    ```bash
    systemctl daemon-reload
    ```

3. Iniciarem el servei:

    ```bash
    systemctl start jupyterhub
    ```

4. Habilitarem el servei:

    ```bash
    systemctl enable jupyterhub
    ```

## Gestio automàtica de comptes d'usuari

En aquest apartat es descriu com donar d'alta usuaris de forma automàtica. Aquesta pràctica és molt útil quan es volen crear molts usuaris. Farem un parell d'eines en bash per a crear usuaris i configurar-los.

### Creació d'usuaris

1. Crearem una carpeta pels scripts.

    ```bash
    mkdir /root/scripts
    ```

2. Afegirem la carpeta de scripts al PATH.

    ```bash
    echo "PATH=$PATH:/root/scripts" >> /root/.bashrc
    ```

3. Crearem un fitxer amb els usuaris que volem crear. En el nostre cas, crearem dos usuaris.

    ```bash
    cat << EOF > /tmp/users.txt
    user1
    user2
    EOF
    ```

4. Crearem un script en bash per a crear els usuaris. Aquest script crearà els usuaris, els directoris de treball i els volums lògics i els assignarà una password aleatoria.

    ```bash
    cat << EOF > /root/scripts/create_users
    #!/bin/bash

    # Creem els usuaris
    while read user; do
        useradd -m -s /bin/bash \$user
    done < /tmp/users.txt

    while read user; do
        lvcreate -L 1G -n \$user users
        mkfs.xfs /dev/users/\$user
        mkdir /home/\$user/notebooks
        mkdir /home/\$user/venv
        echo "/dev/users/\$user /home/\$user/notebooks xfs defaults 0 0" >> /etc/fstab
        mount -a
    done < /tmp/users.txt

    # Assignem una password aleatoria
    while read user; do
        password=\$(openssl rand -base64 12)
        echo \$user:\$password | chpasswd
        echo "Usuari: \$user"
        echo "Password: \$password"
    done < /tmp/users.txt

    # Assignem al grup jupyterhub
    while read user; do
        usermod -a -G jupyterhub \$user
    done < /tmp/users.txt
    ```

5. Crearem un script per eliminar un usuari i tots els fitxers i directoris associats així com el seu volum lògic associat.

    ```bash
    cat << EOF > /tmp/delete_users
    #!/bin/bash

    # Eliminem els usuaris
    while read user; do
        userdel -r \$user
    done < /tmp/users.txt

    # Eliminem els volums lògics
    while read user; do
        lvremove -f /dev/users/\$user
    done < /tmp/users.txt
    ```

6. Per a crear els usuaris, executarem el següent script:

    ```bash
    chmod +x /root/scripts/create_users
    create_users
    ```

## Configuració del tallafocs

1. Instal·larem el paquet `firewalld`:

    ```bash
    dnf install firewalld -y
    ```

2. Iniciarem el servei:

    ```bash
    systemctl start firewalld
    ```

3. Habilitarem el servei:

    ```bash
    systemctl enable firewalld
    ```

4. Crearem una zona per al servei JupyterHub:

    ```bash
    firewall-cmd --permanent --new-zone=jupyterhub
    ```

5. Afegirem el servei JupyterHub a la zona jupyterhub:

    ```bash
    firewall-cmd --permanent --zone=jupyterhub --add-service=http
    ```

6. Recarregarem la configuració del tallafocs:

    ```bash
    firewall-cmd --reload
    ```

## Configuració avanzada de JupyterHub

La configuració de JupyterHub es fa mitjançant un fitxer de configuració que generalment es diu **jupyterhub_config.py**. Aquest fitxer es pot trobar a la carpeta on s'executa JupyterHub o es pot especificar amb la variable d'entorn JUPYTERHUB_CONFIG.

```python
import os

# Configuració del spawner
c.JupyterHub.spawner_class = 'sudospawner.SudoSpawner'
c.SudoSpawner.sudospawner_path = '/opt/jupyterhub/venv/bin/sudospawner'

# Definim el punt d'entrada del servei
c.baseUrl = '/jupyter'

# Configuració del servei de logs
c.JupyterHub.extra_log_file = '/opt/jupyterhub/logs/jupyterhub.log'

# Configuració de l'accés de l'usuari admin
c.Authenticator.admin_users = set()  # Deixa-ho com a conjunt buit per desactivar l'accés de l'usuari admin

# Configuració dels volums i directoris
c.Spawner.notebook_dir = '/home/{username}/notebooks'
```

Per utilitzar aquest fitxer de configuració, caldrà modificar la comanda **ExecStart** del fitxer de configuració del servei JupyterHub:

```bash
ExecStart=/bin/bash -c "/opt/jupyterhub/venv/bin/jupyterhub --config=/opt/jupyterhub/service/jupyterhub_config.py"
```

Finalment haureu de reiniciar el servei JupyterHub.

## Configuració dels usaris a jupytehub (aïllament)

Crearem una configuració que permetra a cada usuari tenir el seu propi entorn virtual i el seu propi kernel. Això permetrà que cada usuari pugui tenir les seves pròpies llibreries i paquets de python.

Per fer-ho, caldrà:

1. Crearem un grup anomenat *estudiants*.

    ```bash
    groupadd estudiants
    ```

2. Únicament els usuaris del grup *estudiants* podran accedir al jupyterhub:

    ```bash
    # Afegir al fitxer de configuració de jupyterhub
    c.Authenticator.whitelist = set()
    c.Authenticator.allowed_groups = {'estudiants'}
    ```

3. Modificar el script de creació d'usuaris per a que creï els entorns virtuals i els kernels.

    ```bash
    cat << EOF > /root/scripts/create_users
    #!/bin/bash

    # Creem els usuaris
    while read user; do
        useradd -m -s /bin/bash \$user
    done < /tmp/users.txt

    while read user; do
        lvcreate -L 1G -n \$user users
        mkfs.xfs /dev/users/\$user
        mkdir /home/\$user/notebooks
        mkdir /home/\$user/venv
        echo "/dev/users/\$user /home/\$user/notebooks xfs defaults 0 0" >> /etc/fstab
        mount -a
    done < /tmp/users.txt

    # Assignem una password aleatoria
    while read user; do
        password=\$(openssl rand -base64 12)
        echo \$user:\$password | chpasswd
        echo "Usuari: \$user"
        echo "Password: \$password"
    done < /tmp/users.txt

    # Assignem al grup jupyterhub
    while read user; do
        usermod -a -G jupyterhub \$user
    done < /tmp/users.txt

    # Crearem els entorns virtuals
    while read user; do
        python3 -m venv /home/\$user/venv
        chown -R \$user:\$user /home/\$user/venv
        echo "source /home/\$user/venv/bin/activate" >> /home/\$user/.bashrc
    done < /tmp/users.txt

    # Crearem el kernel i l'enllaç simbòlic
    while read user; do
        su - \$user -c "source /home/\$user/venv/bin/activate && python3 -m ipykernel install --user --name \$user"
        ln -s /home/\$user/.local/share/jupyter/kernels/\$user /opt/jupyterhub/venv/share/jupyter/kernels/\$user-venv
    done < /tmp/users.txt

    # Afegirem al grup estudiants
    while read user; do
        usermod -a -G estudiants \$user
    done < /tmp/users.txt
    ```

4. Afegirem que cada usuari únicament pugui veure el seu propi kernel:

    ```bash
    c.Spawner.venv_user = True

    import pwd
    import grp

    # Specify the group name
    group_name = 'estudiants'

    # Get a list of all users in the specified group
    group_users = [user.pw_name for user in pwd.getpwall() if group_name in grp.getgrgid(user.pw_gid).gr_name]

    # Configure user_kernel_specs for all users in the group
    c.Spawner.user_kernel_specs = {
        user: {
            'kernel_name': f'{user}-venv',
            'display_name': f'{user} venv',
            'language': 'python',
            'path': f'/opt/jupyterhub/share/jupyter/kernels/{user}-venv',
            'user': user,
        }
        for user in group_users
    }
    ```

    o bé:

    ```bash
    # Afegir al fitxer de configuració de jupyterhub
    c.Spawner.env_keep.append('KERNEL_USERNAME')
    c.Spawner.args = ['--KernelSpecManager.whitelist="*"']
    ```
