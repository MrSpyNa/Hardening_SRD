# Hardening — Seguridad en Redes y Datos

Este repositorio contiene playbooks y roles de Ansible para aplicar un baseline de hardening sobre un servidor GNU/Linux Debian 13.

---

## Arquitectura de prueba

La ejecución fue validada en un entorno de laboratorio con dos máquinas virtuales:

| Rol                  | Sistema    | Descripción                                     |
| -------------------- | ---------- | ----------------------------------------------- |
| Controladora Ansible | Linux Mint | Máquina desde la cual se ejecutan los playbooks |
| Servidor objetivo    | Debian 13  | Máquina sobre la cual se aplica el hardening           |

Direcciones utilizadas en laboratorio:

| Equipo                       | IP               |
| ---------------------------- | ---------------- |
| Linux Mint / Ansible Control | `192.168.56.101` |
| Debian 13 Test               | `192.168.56.10`  |
| Wazuh Manager                | `192.168.56.50`  |

> Las direcciones IP pueden modificarse según el entorno de prueba.

---

## Requisitos previos

### En la máquina desde la que se ejecuta Ansible

Instalar Ansible y herramientas necesarias:

```bash
sudo apt update
sudo apt install -y ansible sshpass git unzip openssh-client
```

Verificar instalación:

```bash
ansible --version
```

### En la máquina Debian 13 objetivo

- Debian 13 Minimal Install / Sin GUI (con SSH + standard system utilities) <br>
    └── Usuario: administrator

El servidor objetivo debe tener instalado:

```bash
sudo apt update
sudo apt install -y sudo openssh-server python3 curl gnupg ufw
```

El servicio SSH debe estar activo:

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh
```

El usuario administrador debe pertenecer al grupo `sudo`:

```bash
sudo usermod -aG sudo administrator
```

---

## Preparación de acceso SSH

Desde la maquina la cual ejecuta Ansible, generar una clave SSH:

```bash
ssh-keygen -t ed25519 -C "ansible-hardening"
```

Copiar la clave al servidor Debian 13:

```bash
ssh-copy-id administrator@192.168.56.10
```

Probar conexión:

```bash
ssh administrator@192.168.56.10
```

---

## Variables principales

Archivo:

```text
group_vars/debian_servers.yml
```

Ejemplo:

```yaml
---
admin_user: administrator
admin_group: sudo

wazuh_manager_ip: "192.168.56.50"

ssh_port: 2222

ssh_allowed_users:
  - "{{ admin_user }}"

system_locale: "en_US.UTF-8"
```

Antes de ejecutar el playbook, ajustar especialmente:

* `admin_user`: usuario administrador permitido por SSH.
* `wazuh_manager_ip`: IP real del servidor Wazuh Manager.
* `ssh_port`: puerto SSH configurado por el hardening.

---

## Ejecución

### 1. Probar conectividad Ansible

```bash
ansible -i inventory/hosts.ini debian_servers -m ping -K
```

El parámetro `-K` solicita la contraseña de `sudo` del usuario remoto.

Resultado esperado:

```text
debian-ese | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

### 2. Ejecutar en modo check

```bash
ansible-playbook -i inventory/hosts.ini site.yml --check --diff -K
```

Este comando permite revisar los cambios que Ansible aplicaría sin modificar el sistema. <br>
**Nota:** Cabe destacar que este comando puede fallar a la hora de probar la instalacion del Wazuh Agent, dado que no se modifican los repositorios y por ende la prueba de instalacion falla.

---

### 3. Ejecutar hardening

```bash
ansible-playbook -i inventory/hosts.ini site.yml --diff -K | tee evidencia-ejecucion-hardening.txt
```

Al finalizar, el servidor queda endurecido y SSH pasa a utilizar el puerto `2222`.

---

## Validación

Cuando Ansible termine de ejecutarse, quedará un archivo de nombre *"evidencia-ejecucion-hardening.txt"*, el cual contiene toda la ejecución del código, además de dar el índice de Lynis, el cual se ejecuta al finalizar el script.

---

## Consideraciones importantes

### Cambio de puerto SSH

El playbook cambia el puerto SSH de `22` a `2222`.
Por eso, la primera ejecución debe realizarse usando `ansible_port=22`, y las ejecuciones posteriores deben usar `ansible_port=2222`.

---

## Estructura del repositorio

```text
ese-hardening/
├── ansible.cfg
├── site.yml
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── debian_servers.yml
└── roles/
    ├── auditd/
    ├── extra_hardening/
    ├── filesystem/
    ├── kernel_parameters/
    ├── rsyslog/
    ├── services/
    ├── ssh_hardening/
    ├── ufw_firewall/
    ├── users_privileges/
    └── wazuh_agent/
```
