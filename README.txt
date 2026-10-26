=====================================================================
 ASO - Tareas 3, 4 y 5 - Grupo 10 "anonymous"
 Despliegue remoto con Ansible (roles) sobre openSUSE 16.0:
 Apache + PHP + phpMyAdmin + MariaDB + Samba + FTP + PostgreSQL
=====================================================================

a) REPOSITORIO GITHUB
---------------------
https://github.com/ErikaVidal/Trabajo-Ansible-Github
(Compartido con gcussi@yahoo.com)

b) PASOS PARA EJECUTAR EL PROYECTO
----------------------------------
Requisitos
  CONTROL (VM con Ansible, openSUSE 16.0):
      zypper in -y ansible sshpass git
    Si falta algun modulo (por ejemplo con ansible-core en otra distro):
      ansible-galaxy collection install community.general community.mysql community.postgresql ansible.posix

  DESTINO (VM openSUSE 16.0 limpia, con red e internet):
    Debe aceptar SSH como root. Si no lo permite, en el DESTINO:
      echo "PermitRootLogin yes" > /etc/ssh/sshd_config.d/root.conf
      systemctl enable --now sshd && systemctl restart sshd

1. Clonar el repositorio en la VM de control
      git clone https://github.com/ErikaVidal/Trabajo-Ansible-Github.git
      cd Trabajo-Ansible-Github

2. Poner la IP del DESTINO en inventory/hosts.yml (campo ansible_host)
   Si PostgreSQL se va a usar desde otra red, ajustar pg_remote_network
   en group_vars/all/vars.yml (por defecto 192.168.0.0/16).

3. Ejecutar el despliegue
      ansible-playbook site.yml -k --ask-vault-pass

   Se piden dos contrasenas:
      SSH password   -> contrasena de root del DESTINO
      Vault password -> se entrega junto a este README

   (Opcional) Sin contrasena SSH, usando una clave:
      ssh-keygen -t ed25519
      ssh-copy-id root@IP_DESTINO
      ansible-playbook site.yml --ask-vault-pass

   (Opcional) Ejecutar solo una parte con tags:
      lamp, apache, mariadb, phpmyadmin, samba, ftp, postgresql
      ansible-playbook site.yml -k --ask-vault-pass --tags postgresql

4. Verificar
      ansible-playbook verify.yml -k

   Desde otra maquina de la red:
      Apache/PHP   http://IP_DESTINO/
      phpMyAdmin   http://IP_DESTINO/phpmyadmin      (usuario aso_admin)
      PHP->PgSQL   http://IP_DESTINO/pgtest.php
      Samba        smbclient //IP_DESTINO/aso_share -U estudiante
                   (Windows: \\IP_DESTINO\aso_share)
      FTP          ftp IP_DESTINO                     (usuario estudiante)
      SFTP         sftp estudiante@IP_DESTINO
      PostgreSQL   psql "host=IP_DESTINO dbname=aso_pgdb user=aso_pguser sslmode=require"

   Ver las contrasenas de los servicios:
      ansible-vault view group_vars/all/vault.yml

Nota: si Ansible avisa que ignora ansible.cfg por estar en un directorio
con permisos de escritura para todos, ejecutar dentro del proyecto: chmod 755 .

ESTRUCTURA
----------
ansible.cfg            Configuracion (pipelining, ControlPersist, forks, tiempos)
inventory/hosts.yml    Servidor destino: solo la IP, sin usuario ni contrasena
group_vars/all/
  vars.yml             Variables comunes (usuarios, bases, red permitida)
  vault.yml            Contrasenas cifradas con Ansible Vault (AES256)
  vault.yml.example    Plantilla del vault sin secretos
site.yml               Playbook principal (llama a los roles)
verify.yml             Comprobacion de servicios y puertos
roles/
  apache_php/          T3  Apache + PHP, SELinux y firewall
  mariadb/             T3  MariaDB: clave root, sin anonimos, base y usuario
  phpmyadmin/          T3  phpMyAdmin publicado en /phpmyadmin
  samba/               T3  usuario de sistema, usuario Samba, directorio,
                           smb.conf con create mask / directory mask
  vsftpd/              T3  FTP con chroot y puertos pasivos
  postgresql/          T5  PostgreSQL + php8-pgsql, clave de postgres,
                           usuario y base nuevos, conexion remota solo SSL

TAREA 4 (GitHub)
----------------
Scripts y plantillas versionados en el repositorio:
  roles/samba/templates/smb.conf.j2
  roles/vsftpd/templates/vsftpd.conf.j2
  roles/phpmyadmin/templates/config.inc.php.j2, phpmyadmin.conf.j2
  roles/apache_php/templates/index.php.j2
  roles/postgresql/templates/pgtest.php.j2

MEJORAS PEDIDAS
---------------
- Host file sin usuario/contrasena visibles: el inventario solo tiene la
  IP; la clave SSH se pide con -k (o se usa clave publica) y los demas
  secretos van cifrados en el vault. Tareas con secretos usan no_log.
- Tiempo de respuesta: pipelining + ControlPersist (menos conexiones
  SSH), forks=10, gather_facts desactivado, paquetes de cada rol en una
  sola transaccion de zypper, sin "zypper update" completo y tareas
  idempotentes (en la 2da ejecucion casi todo sale "ok"). El callback
  profile_tasks muestra cuanto tarda cada tarea.
- Roles independientes, ejecutables por separado con --tags.

PostgreSQL: cifrado y acceso remoto
-----------------------------------
- listen_addresses = '*' y ssl = on (certificado autofirmado generado
  en el destino), contrasenas con scram-sha-256.
- pg_hba.conf: "hostssl" para la red permitida y "hostnossl ... reject",
  asi una conexion remota sin SSL es rechazada.
