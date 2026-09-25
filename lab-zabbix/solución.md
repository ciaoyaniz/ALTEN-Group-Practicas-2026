# Solución

> [!NOTE] Capturas
> Todas las capturas van en la carpeta `lab-zabbix/capturas/`. Cada hueco está marcado con `📷 CAPTURA` y el nombre de archivo esperado. Si pegas la imagen desde Obsidian (queda como `Pasted image ...png`), basta con sustituir el enlace.

> [!NOTE] Base
> Este laboratorio sigue el manual *Zabbix de Cero a Experto - Laboratorio Multiagente*: las VM Linux y la de Windows se crean con **Vagrant** (PASO 4 y PASO 7).

## Arquitectura

El Zabbix Server corre en **Docker** sobre mi PC con Windows, y los hosts monitorizados son **VM de VirtualBox**, todas en **red puente** (`192.168.1.0/24`) para que se vean con el host:

| # | Host | Rol / Servicio | Cómo se crea | IP | Agente |
|---|---|---|---|---|---|
| 1 | `zbx-server` (PC) | Zabbix Server + MySQL + Frontend | Docker Compose | `.100` | — |
| 2 | `zbx-agent-1` | Linux base (CPU/RAM/disco) | Vagrant | `.101` | Agent 2, activo |
| 3 | `zbx-agent-2` | Nginx | Vagrant | `.102` | Agent 2, activo |
| 4 | `zbx-agent-3` | MySQL | Vagrant | `.103` | Agent 2, activo |
| 5 | `zbx-agent-4` | RabbitMQ (HTTP Agent) | Vagrant | `.104` | Agent 2, activo |
| 6 | `zbx-agent-5` | OpenIndiana / Solaris | Manual | `.105` | Agente clásico, activo |
| 7 | `zbx-agent-6` | Windows Server, AD DS, LDAP | Vagrant (WinRM) | `.106` | Agent 2 Windows, activo |
| 8 | `zbx-agent-7` | PostgreSQL | Vagrant | `.107` | Agent 2, activo |
| 9 | `zbx-agent-8` | Redis | Vagrant | `.108` | Agent 2, activo |
| 10 | `zbx-agent-9` | Docker host | Vagrant | `.109` | Agent 2, activo |
| 11 | `zbx-agent-10` | SNMP + Zabbix Proxy | Vagrant | `.110` | Agent 2, activo + proxy |

📷 CAPTURA: `capturas/01-arquitectura.png` (diagrama o vista de VirtualBox con todas las VM)

```
192.168.1.0/24
├── .1        Router / gateway
├── .2–.99    DHCP del router (resto de dispositivos)
├── .100      PC Windows (Docker: Zabbix Server + web + MySQL)
├── .101–.110 Agentes del laboratorio
└── .111–.254 Libre
```

Especificaciones de cada VM:

| Host | RAM | CPU | Disco | Sistema |
|---|---|---|---|---|
| Agentes Linux | 1–1,5 GB | 1 | 10 GB | Ubuntu Server 24.04 LTS (`bento/ubuntu-24.04`) |
| `zbx-agent-5` | 2 GB | 1 | 20 GB | OpenIndiana Hipster |
| `zbx-agent-6` | 3 GB | 2 | 40 GB | Windows Server 2022 Evaluation |

Estructura de la carpeta:

```
lab-zabbix/
├── zabbix-docker/
│   └── docker-compose.yml     # Zabbix Server + web + MySQL (+ Grafana)
├── zabbix-vagrant/
│   ├── Vagrantfile            # 8 VM Linux
│   └── provision/*.sh         # agente común + servicio de cada rol
├── zabbix-vagrant-windows/
│   ├── Vagrantfile            # VM Windows Server
│   └── provision/install-agent2.ps1
├── capturas/
├── practica.md
└── solución.md
```

> El `.gitignore` del repo solo sube los `.md` y las capturas, así que copio aquí el contenido de los archivos importantes.

### Por qué funciona con red puente

Docker Desktop con backend **WSL2** publica los puertos de los contenedores en **todas las interfaces** del PC, no solo en `localhost`. Así, el `8080` (frontend) y el `10051` (server) quedan accesibles en `192.168.1.100` desde cualquier VM en modo puente, igual que si Zabbix estuviera instalado directamente en el PC.

### Conceptos básicos

- **Zabbix Server**: recibe los datos, evalúa triggers, lanza acciones y escribe en la base de datos.
- **Base de datos**: MySQL 8 en este laboratorio. Guarda configuración, historial y tendencias.
- **Frontend**: web PHP (Nginx) donde se configura todo y se ven los datos.
- **Agente**: recoge métricas del equipo y las envía (o las expone) al server.
- **Proxy**: recoge datos de una red remota y los reenvía en lotes al server.

| | Agente clásico (`zabbix-agent`) | Zabbix Agent 2 (`zabbix-agent2`) |
|---|---|---|
| Lenguaje | C | Go |
| Plataformas | Linux, **Solaris**, AIX, FreeBSD, Windows… | Linux, Windows, macOS (**no Solaris**) |
| Plugins nativos | No (UserParameters) | Sí (MySQL, PostgreSQL, Redis, Docker…) |
| En este lab | Solo `zbx-agent-5` | Los otros 9 agentes |

**Modo pasivo**: el server se conecta al agente (puerto `10050`). **Modo activo**: el agente se conecta al server (puerto `10051`), pide su lista de checks y envía los datos. Uso **modo activo en todos los hosts**: solo hay que abrir el `10051` en el PC, una vez.

Los tres parámetros clave del agente:

```
Server=192.168.1.100          # quién puede hacer consultas pasivas
ServerActive=192.168.1.100    # a quién envía los datos (modo activo)
Hostname=zbx-agent-1          # debe coincidir EXACTAMENTE con el host del frontend
```

- **Item**: una métrica (`system.cpu.load`). **Trigger**: expresión que define un problema. **Template**: conjunto reutilizable de items, triggers y gráficos. **Action**: qué hacer cuando ocurre algo (email, Telegram…).
- **Historial** (valores crudos, 31 días por defecto) y **tendencias** (agregados por hora, 365 días por defecto).

## PASO 1: Preparar el anfitrión (Windows)

### Virtualización y WSL2

Activo **Intel VT-x / AMD SVM** en la BIOS y lo compruebo en PowerShell como administrador:

```powershell
Get-ComputerInfo -Property "HyperVRequirementVirtualizationFirmwareEnabled"   # True
wsl --install        # instala WSL2 + Ubuntu
wsl --status         # Versión predeterminada: 2
```

### Docker Desktop, VirtualBox y Vagrant

- **Docker Desktop**, con *Use the WSL 2 based engine* activado.
- **VirtualBox 7.x**: convive con Hyper-V/WSL2, no hace falta desactivarlo.

```powershell
docker --version
docker compose version
vagrant --version
```

📷 CAPTURA: `capturas/02-requisitos-versiones.png`

### Vagrant

Instalo Vagrant desde `https://developer.hashicorp.com/vagrant/install` y el plugin `vagrant-vbguest`:

```powershell
vagrant --version
vagrant plugin install vagrant-vbguest
```

### IP fija del PC

Todos los agentes apuntan a `192.168.1.100`, así que el PC necesita esa IP siempre. La fijé en *Configuración > Red e Internet > Asignación de IP > Manual*, y comprobé que el rango `.100`–`.110` queda **fuera del DHCP** del router.

```powershell
ipconfig
```

📷 CAPTURA: `capturas/03-ip-fija-host.png`

## PASO 2: Zabbix Server con Docker Compose

### El `docker-compose.yml`

```yaml
services:
  mysql-server:
    image: mysql:8.0
    container_name: zabbix-mysql
    restart: unless-stopped
    command:
      - mysqld
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_bin
      - --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_pwd
      MYSQL_ROOT_PASSWORD: root_pwd
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - zbx-net

  zabbix-server:
    image: zabbix/zabbix-server-mysql:ubuntu-7.0-latest
    container_name: zabbix-server
    restart: unless-stopped
    environment:
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_pwd
      MYSQL_ROOT_PASSWORD: root_pwd
    ports:
      - "10051:10051"
    depends_on:
      - mysql-server
    networks:
      - zbx-net

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:ubuntu-7.0-latest
    container_name: zabbix-web
    restart: unless-stopped
    environment:
      ZBX_SERVER_HOST: zabbix-server
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_pwd
      MYSQL_ROOT_PASSWORD: root_pwd
      PHP_TZ: Europe/Madrid
    ports:
      - "8080:8080"
    depends_on:
      - zabbix-server
      - mysql-server
    networks:
      - zbx-net

volumes:
  mysql-data:

networks:
  zbx-net:
    driver: bridge
```

- `mysql-server`: la base de datos de Zabbix. `utf8mb4_bin` es el cotejamiento que exige Zabbix. Los datos se guardan en el volumen `mysql-data`, así no se pierden al recrear el contenedor.
- `zabbix-server`: el proceso que recibe y procesa los datos. Publica el `10051`, que es donde se conectan los agentes activos.
- `zabbix-web`: el frontend (Nginx + PHP), publicado en el `8080`.
- `zbx-net`: red interna de Docker. Dentro de ella los contenedores se encuentran por nombre (`mysql-server`, `zabbix-server`), por eso no hace falta poner IPs.
- `depends_on`: solo controla el **orden de arranque**, no espera a que MySQL esté listo. Por eso el server puede reintentar la conexión unos segundos al principio.

### Levantar el stack

```powershell
cd zabbix-docker
docker compose up -d
docker compose ps
```

📷 CAPTURA: `capturas/04-docker-compose-ps.png` (los tres contenedores `Up`)

La primera vez el server crea el esquema de la base de datos, lo compruebo en los logs:

```powershell
docker compose logs -f zabbix-server
```

📷 CAPTURA: `capturas/05-logs-zabbix-server.png`

Si `zabbix-server` se reinicia en bucle, normalmente es que MySQL aún no ha terminado de inicializarse: espero 1-2 minutos y vuelvo a mirar los logs.

### Comprobar que el 10051 es accesible desde la LAN

Pruebo contra la IP del PC, no contra `localhost`:

```powershell
Test-NetConnection -ComputerName 192.168.1.100 -Port 10051
```

`TcpTestSucceeded` debe ser `True`.

📷 CAPTURA: `capturas/06-test-netconnection.png`

### Abrir el puerto 10051 en el firewall de Windows

Los agentes en modo activo se conectan al `10051` del PC. En PowerShell como administrador:

```powershell
New-NetFirewallRule -DisplayName "Zabbix Server 10051" -Direction Inbound -Protocol TCP -LocalPort 10051 -Action Allow
```

(Opcional: lo mismo con el `8080` si quiero abrir el frontend desde otro equipo de la red.)

📷 CAPTURA: `capturas/07-firewall-10051.png`

## PASO 3: Frontend de Zabbix

### Primer login

Entro en `http://localhost:8080` con el usuario por defecto `Admin` / `zabbix`.

📷 CAPTURA: `capturas/08-login-zabbix.png`

### Cambiar la contraseña, idioma y zona horaria

Icono de usuario > **Profile**:

- **Change password**: pongo una contraseña nueva (nunca dejar la de por defecto).
- **Language**: `Español (es_ES)`.
- **Time zone**: `Europe/Madrid`.

Compruebo la versión en el pie de la pantalla de login o en **Help > About**: debe ser **7.0.x**.

📷 CAPTURA: `capturas/09-perfil-admin.png`

### Recuperar la contraseña de Admin desde MySQL

Por si alguna vez pierdo el acceso. Zabbix 7.0 guarda la contraseña con **bcrypt**, así que genero un hash nuevo con Python:

```powershell
pip install bcrypt
python -c "import bcrypt; print(bcrypt.hashpw(b'NuevaPassword123', bcrypt.gensalt(10)).decode())"
```

Y lo aplico dentro del contenedor de MySQL:

```powershell
docker exec -it zabbix-mysql mysql -uroot -p
```

```sql
USE zabbix;
SELECT userid, username FROM users WHERE username = 'Admin';
UPDATE users SET passwd = '<hash-generado>' WHERE username = 'Admin';
UPDATE users SET attempt_failed = 0 WHERE username = 'Admin';   -- por si quedó bloqueado
EXIT;
```

> [!warning] Siempre con `WHERE`
> Un `UPDATE` sin `WHERE` sobre `users` cambiaría la contraseña de todas las cuentas.

También creé una cuenta de respaldo `admin-backup` con rol **Super Admin** (*Users > Users > Create user*).

📷 CAPTURA: `capturas/10-reset-password-mysql.png`

### Recorrido por la interfaz

- **Monitoring**: consumo de datos (Dashboards, Problems, Hosts, Latest data, Maps, Discovery).
- **Data collection**: configuración (Host groups, Hosts, Templates, Maintenance, Discovery, Actions).
- **Alerts**: Media types, Actions, Scripts.
- **Reports**: disponibilidad, top triggers, auditoría, notificaciones.
- **Users**: usuarios, grupos, roles, autenticación (LDAP).
- **Administration**: General, Proxies, Queue, Housekeeping.

📷 CAPTURA: `capturas/11-dashboard-inicial.png`

## PASO 4: Levantar los agentes Linux con Vagrant

### Por qué Vagrant

Vagrant describe las VM **como código** en un `Vagrantfile`: descarga la imagen base, crea las VM en VirtualBox, les pone red e IP y ejecuta scripts de *provisioning* que instalan el agente y el servicio. Con un `vagrant up` se levantan las 8 VM ya configuradas, sin abrir VirtualBox ni instalar Ubuntu a mano.

### El `Vagrantfile`

```ruby
IP_BASE = "192.168.1"          # ajustá al rango de tu propia red (03 – Diseño de la infraestructura)
ZABBIX_SERVER_IP = "192.168.1.100"

AGENTS = [
  { name: "zbx-agent-1", ip: 101, role: "base" },
  { name: "zbx-agent-2", ip: 102, role: "nginx" },
  { name: "zbx-agent-3", ip: 103, role: "mysql" },
  { name: "zbx-agent-4", ip: 104, role: "rabbitmq" },
  { name: "zbx-agent-7", ip: 107, role: "postgresql" },
  { name: "zbx-agent-8", ip: 108, role: "redis" },
  { name: "zbx-agent-9", ip: 109, role: "docker" },
  { name: "zbx-agent-10", ip: 110, role: "snmp-proxy" },
]

Vagrant.configure("2") do |config|
  config.vbguest.auto_update = false
  AGENTS.each do |agent|
    config.vm.define agent[:name] do |node|
      node.vm.box = "bento/ubuntu-24.04"
      node.vm.hostname = agent[:name]

      # Red puente: la VM queda en la misma LAN que el host Windows
      node.vm.network "public_network", ip: "#{IP_BASE}.#{agent[:ip]}"

      node.vm.provider "virtualbox" do |vb|
        vb.name = agent[:name]
        vb.memory = agent[:role] == "mysql" || agent[:role] == "postgresql" ? 1536 : 1024
        vb.cpus = 1
      end

      # Provisioning común: instala y configura Zabbix Agent 2
      node.vm.provision "shell", path: "provision/common-agent.sh",
        args: [ZABBIX_SERVER_IP, agent[:name]]

      # Provisioning específico del servicio de este host
      node.vm.provision "shell", path: "provision/#{agent[:role]}.sh"
    end
  end
end```

- La lista `AGENTS` es la única fuente de verdad: añadir o quitar un host es añadir o quitar una línea.
- `bento/ubuntu-24.04`: imagen base de Ubuntu 24.04 para VirtualBox.
- `public_network` con `ip:` = adaptador **puente** con IP fija.
- A las VM de bases de datos les doy más RAM (1536 MB).
- `config.vbguest.auto_update = false`: evita que el plugin `vagrant-vbguest` intente compilar las Guest Additions (falla con *linux-headers… has no installation candidate* y no las necesitamos).
- Cada VM ejecuta **dos scripts**: el común (Agent 2) y el de su rol.

### Script común: `provision/common-agent.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

ZABBIX_SERVER_IP="$1"
AGENT_HOSTNAME="$2"

# Repositorio oficial de Zabbix 7.0 para Ubuntu 24.04
wget -q https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb -O /tmp/zabbix-release.deb
dpkg -i /tmp/zabbix-release.deb
apt-get update -qq

apt-get install -y zabbix-agent2 zabbix-agent2-plugin-postgresql zabbix-agent2-plugin-mongodb

# Configuración: modo activo apuntando al server Docker en el host Windows
sed -i "s/^Server=.*/Server=${ZABBIX_SERVER_IP}/" /etc/zabbix/zabbix_agent2.conf
sed -i "s/^ServerActive=.*/ServerActive=${ZABBIX_SERVER_IP}/" /etc/zabbix/zabbix_agent2.conf
sed -i "s/^Hostname=.*/Hostname=${AGENT_HOSTNAME}/" /etc/zabbix/zabbix_agent2.conf

systemctl restart zabbix-agent2
systemctl enable zabbix-agent2

# Firewall interno de la VM (ufw), por si está activo
if command -v ufw >/dev/null && ufw status | grep -q "Status: active"; then
  ufw allow out to "${ZABBIX_SERVER_IP}" port 10051 proto tcp
fi

echo "Agent 2 configurado en ${AGENT_HOSTNAME}, apuntando a ${ZABBIX_SERVER_IP}"```

- Añade el repositorio oficial de Zabbix 7.0 e instala Agent 2 con los plugins de PostgreSQL y MongoDB. El de MySQL ya viene dentro de `zabbix-agent2` (no existe el paquete `zabbix-agent2-plugin-mysql`).
- Con `sed` pone `Server`, `ServerActive` y `Hostname` (el nombre de la VM).

### Scripts de cada rol

| Script | Qué hace |
|---|---|
| `base.sh` | Nada, host de referencia |
| `nginx.sh` | Instala y habilita Nginx |
| `mysql.sh` | Instala y habilita MySQL |
| `rabbitmq.sh` | Repositorio oficial de RabbitMQ, instala `rabbitmq-server`, activa `rabbitmq_management` y crea el usuario `zbx_monitor` (tag `monitoring`) |
| `postgresql.sh` | Instala y habilita PostgreSQL |
| `redis.sh` | Instala Redis con `requirepass MonitorPass123!` y `enable-debug-command local` |
| `docker.sh` | Instala Docker Engine (repo oficial) y mete al usuario `zabbix` en el grupo `docker` |
| `snmp-proxy.sh` | Instala `snmpd` (comunidad `public`) y Zabbix Proxy con SQLite (`/var/lib/zabbix/zabbix_proxy.db`) |

Estructura:

```
zabbix-vagrant/
├── Vagrantfile
└── provision/
    ├── common-agent.sh
    ├── base.sh  nginx.sh  mysql.sh  rabbitmq.sh
    ├── postgresql.sh  redis.sh  docker.sh
    └── snmp-proxy.sh
```

### Levantar las VM

```powershell
cd D:\Estudio\Laboratorios\ALTEN-Group-Practicas-2026\lab-zabbix\zabbix-vagrant
vagrant validate
vagrant up                  # las 8
vagrant up zbx-agent-3      # o una sola
vagrant status
```

La primera vez descarga la box `bento/ubuntu-24.04` (~600 MB); las demás VM la reutilizan. Al usar red puente, Vagrant pregunta qué adaptador usar: elijo el que tiene salida a internet.

📷 CAPTURA: `capturas/12-vagrant-up.png`

📷 CAPTURA: `capturas/13-vagrant-status.png`

### Comprobar cada VM

```powershell
vagrant ssh zbx-agent-1
```

```bash
sudo systemctl status zabbix-agent2 --no-pager
grep -E '^(Server|ServerActive|Hostname)=' /etc/zabbix/zabbix_agent2.conf
sudo tail -20 /var/log/zabbix/zabbix_agent2.log
ip a | grep "inet 192.168"
ping -c 3 192.168.1.100
exit
```

📷 CAPTURA: `capturas/14-agent2-status.png`

### Comandos de ciclo de vida

```powershell
vagrant provision zbx-agent-2   # vuelve a ejecutar los scripts (si los edito)
vagrant halt                    # apaga todas (libera RAM sin perder nada)
vagrant halt zbx-agent-3        # apaga una
vagrant reload                  # reinicia aplicando cambios del Vagrantfile
vagrant destroy -f zbx-agent-3  # borra la VM (sin confirmación)
```

No hace falta tenerlas todas encendidas: `vagrant up <host>` antes de cada paso y `vagrant halt` al terminar.

## PASO 5: Alta de hosts y verificación

### Host groups

*Data collection > Host groups > Create host group*: `Linux servers - Lab`, `Databases - Lab`, `Messaging - Lab`, `Web servers - Lab`, `Windows servers - Lab`, `Network devices - Lab`.

### Crear los hosts

*Data collection > Hosts > Create host*:

- **Host name**: exactamente el `Hostname` del agente.
- **Host groups**: los que correspondan (un host puede estar en varios).
- **Templates**: `Linux by Zabbix agent active` (la variante **active**, coherente con el modo activo).
- **Interfaces**: *Agent*, IP de la VM, puerto `10050` (en modo activo no se usa para consultar, pero Zabbix la pide como referencia).

| Host | Host groups | Template |
|---|---|---|
| `zbx-agent-1` | Linux servers - Lab | Linux by Zabbix agent active |
| `zbx-agent-2` | Linux servers - Lab, Web servers - Lab | Linux by Zabbix agent active |
| `zbx-agent-3` | Linux servers - Lab, Databases - Lab | Linux by Zabbix agent active |
| `zbx-agent-4` | Linux servers - Lab, Messaging - Lab | Linux by Zabbix agent active |
| `zbx-agent-7` | Linux servers - Lab, Databases - Lab | Linux by Zabbix agent active |
| `zbx-agent-8` | Linux servers - Lab, Databases - Lab | Linux by Zabbix agent active |
| `zbx-agent-9` | Linux servers - Lab | Linux by Zabbix agent active |
| `zbx-agent-10` | Linux servers - Lab | Linux by Zabbix agent active |

Los templates de cada servicio se añaden en sus pasos (PASO 15 en adelante).

📷 CAPTURA: `capturas/15-create-host.png`

### Activar el host "Zabbix server"

Viene precargado para monitorizar el propio server. Lo activo si está *Disabled* y dejo su interfaz en `127.0.0.1` (dentro del contenedor).

### Verificar

En *Data collection > Hosts*, el icono **ZBX** de cada host:

- **Verde**: llegan datos.
- **Rojo**: sin conexión (agente parado, firewall, hostname mal escrito, IP incorrecta).
- **Gris**: sin actividad o template pasivo en lugar de activo.

Pasando el ratón por encima se ve el error exacto. Después compruebo en *Monitoring > Latest data* que los valores se actualizan.

📷 CAPTURA: `capturas/16-lista-hosts.png`

📷 CAPTURA: `capturas/17-latest-data.png`

## PASO 6: OpenIndiana / Solaris (`zbx-agent-5`, manual)

Agent 2 no existe para Solaris/illumos, así que aquí uso el **agente clásico** compilado desde el código fuente, y la VM se instala a mano.

### Crear la VM e instalar

- ISO **OpenIndiana Hipster** (edición de texto) de `https://www.openindiana.org/downloads/`.
- VirtualBox: tipo *Oracle Solaris 11 64-bit*, 2048 MB, 1 CPU, disco de 20 GB, **adaptador puente**.
- Instalador de texto: idioma, teclado, disco completo, zona horaria y un usuario (`zbxuser`) con `sudo`.

📷 CAPTURA: `capturas/18-openindiana-instalacion.png`

### IP fija con `ipadm`

OpenIndiana no usa Netplan:

```bash
ipadm show-if
sudo ipadm create-addr -T static -a 192.168.1.105/24 net0/v4
sudo route -p add default 192.168.1.1
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
ipadm show-addr
ping 192.168.1.100
```

### Compilar el agente clásico

```bash
sudo pkg install developer/gcc developer/build/gnu-make developer/build/autoconf developer/build/automake developer/build/libtool
curl -O https://cdn.zabbix.com/zabbix/sources/stable/7.0/zabbix-7.0.0.tar.gz
tar -xzf zabbix-7.0.0.tar.gz && cd zabbix-7.0.0
./configure --enable-agent --prefix=/usr/local/zabbix
gmake
sudo gmake install
```

Solo compilo el agente (`--enable-agent`), con la misma versión mayor que el server.

`/usr/local/zabbix/etc/zabbix_agentd.conf`:

```
Server=192.168.1.100
ServerActive=192.168.1.100:10051
Hostname=zbx-agent-5
RefreshActiveChecks=60
```

📷 CAPTURA: `capturas/19-openindiana-compilacion.png`

### Servicio SMF

OpenIndiana usa **SMF** en vez de systemd. Creo el manifiesto:

```bash
sudo tee /var/svc/manifest/network/zabbix-agent.xml > /dev/null <<'XML'
<?xml version="1.0"?>
<!DOCTYPE service_bundle SYSTEM "/usr/share/lib/xml/dtd/service_bundle.dtd.1">
<service_bundle type="manifest" name="zabbix-agent">
  <service name="network/zabbix-agent" type="service" version="1">
    <create_default_instance enabled="true"/>
    <single_instance/>
    <dependency name="network" grouping="require_all" restart_on="error" type="service">
      <service_fmri value="svc:/milestone/network:default"/>
    </dependency>
    <exec_method type="method" name="start"
      exec="/usr/local/zabbix/sbin/zabbix_agentd -c /usr/local/zabbix/etc/zabbix_agentd.conf"
      timeout_seconds="60"/>
    <exec_method type="method" name="stop" exec=":kill" timeout_seconds="60"/>
    <stability value="Evolving"/>
  </service>
</service_bundle>
XML

sudo svccfg import /var/svc/manifest/network/zabbix-agent.xml
sudo svcadm enable zabbix-agent
svcs zabbix-agent          # online
```

📷 CAPTURA: `capturas/20-openindiana-svcs.png`

### Alta en el frontend

- **Host name**: `zbx-agent-5`, grupo `Linux servers - Lab` (o `Unix servers - Lab`).
- **Template**: **`Zabbix agent active`** (genérico). `Linux by Zabbix agent` no sirve: usa items de `/proc` que Solaris no tiene.
- Interfaz Agent `192.168.1.105:10050`.
- Items extra opcionales (tipo *Zabbix agent (active)*): `system.uptime`, `system.cpu.load[all,avg1]`, `vfs.fs.size[/,pfree]`.

📷 CAPTURA: `capturas/21-openindiana-host-verde.png`

## PASO 7: Windows Server con Vagrant (`zbx-agent-6`)

### `zabbix-vagrant-windows/Vagrantfile`

```ruby
Vagrant.configure("2") do |config|
  config.vm.define "zbx-agent-6" do |node|
    node.vm.box = "gusztavvargadr/windows-server"
    node.vm.hostname = "zbx-agent-6"

    node.vm.network "public_network", ip: "192.168.1.106"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "zbx-agent-6"
      vb.memory = 3072
      vb.cpus = 2
    end

    node.vm.communicator = "winrm"
    node.winrm.username = "vagrant"
    node.winrm.password = "vagrant"

    node.vm.provision "shell", path: "provision/install-agent2.ps1",
      args: ["192.168.1.100", "zbx-agent-6"]
  end
end```

Proyecto separado porque Windows usa **WinRM** en lugar de SSH. Le doy 3 GB y 2 CPU.

### `provision/install-agent2.ps1`

```powershell
param(
  [string]$ZabbixServerIP,
  [string]$AgentHostname
)

$ErrorActionPreference = "Stop"

# Descargar Zabbix Agent 2 para Windows (MSI oficial, 7.0 LTS)
$msiUrl = "https://cdn.zabbix.com/zabbix/binaries/stable/7.0/7.0.0/zabbix_agent2-7.0.0-windows-amd64-openssl.msi"
$msiPath = "$env:TEMP\zabbix_agent2.msi"
Invoke-WebRequest -Uri $msiUrl -OutFile $msiPath

Start-Process msiexec.exe -ArgumentList "/i `"$msiPath`" /quiet SERVER=$ZabbixServerIP SERVERACTIVE=$ZabbixServerIP HOSTNAME=$AgentHostname" -Wait

Start-Service "Zabbix Agent 2"
Set-Service "Zabbix Agent 2" -StartupType Automatic

# Abrir el firewall de Windows para el tráfico saliente hacia el server (modo activo)
New-NetFirewallRule -DisplayName "Zabbix Agent 2 outbound" -Direction Outbound -Protocol TCP -RemotePort 10051 -Action Allow

Write-Output "Zabbix Agent 2 instalado y configurado en $AgentHostname"```

El MSI acepta `SERVER`, `SERVERACTIVE` y `HOSTNAME`, así que la instalación es desatendida.

```powershell
cd D:\Estudio\Laboratorios\ALTEN-Group-Practicas-2026\lab-zabbix\zabbix-vagrant-windows
vagrant up
vagrant winrm zbx-agent-6 -c "Get-Service 'Zabbix Agent 2'"
```

La primera vez tarda 10-20 minutos (la box pesa varios GB y hay que esperar a WinRM).

Alta en el frontend: `zbx-agent-6`, grupo `Windows servers - Lab`, template `Windows by Zabbix agent active`, interfaz `192.168.1.106:10050`.

📷 CAPTURA: `capturas/22-vagrant-up-windows.png`

📷 CAPTURA: `capturas/23-servicio-agent2-windows.png`

### Promover a Domain Controller

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

Install-ADDSForest `
  -DomainName "lab.zabbix.local" `
  -DomainNetbiosName "LAB" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rdSeguro!" -AsPlainText -Force) `
  -Force:$true
```

La VM se reinicia sola al terminar. Después:

```powershell
Get-ADDomain

New-ADUser -Name "Zabbix LDAP User" -SamAccountName "zbxldap" `
  -UserPrincipalName "zbxldap@lab.zabbix.local" `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rdSeguro!" -AsPlainText -Force) `
  -Enabled $true
```

📷 CAPTURA: `capturas/24-get-addomain.png`

### Autenticación LDAP en Zabbix

*Users > Authentication > LDAP settings* → *Enable LDAP authentication* → *Add*:

- Host `192.168.1.106`, puerto `389`.
- Base DN `dc=lab,dc=zabbix,dc=local`.
- Search attribute `sAMAccountName`.
- Bind DN / password: una cuenta con lectura del directorio (para probar, `zbxldap`).

Con **Test** valido el usuario `zbxldap`. Luego creo en *Users > Users* un usuario `zbxldap` (sin contraseña local) para que entre con la del dominio.

📷 CAPTURA: `capturas/25-ldap-test.png`

## PASO 8: Cifrado PSK

Por defecto el tráfico agente-server va **sin cifrar ni autenticar**. Con **PSK** (clave compartida) lo cifro sin necesitar certificados. Lo aplico en `zbx-agent-1` y `zbx-agent-6`, con una clave **distinta** por host.

### Linux (`zbx-agent-1`)

```bash
sudo sh -c 'openssl rand -hex 32 > /etc/zabbix/zbx-agent-1.psk'
sudo chown zabbix:zabbix /etc/zabbix/zbx-agent-1.psk
sudo chmod 400 /etc/zabbix/zbx-agent-1.psk
sudo cat /etc/zabbix/zbx-agent-1.psk        # la copio para el frontend
```

Añado a `/etc/zabbix/zabbix_agent2.conf`:

```
TLSConnect=psk
TLSPSKIdentity=PSK-zbx-agent-1
TLSPSKFile=/etc/zabbix/zbx-agent-1.psk
```

```bash
sudo systemctl restart zabbix-agent2
```

En el frontend, *Hosts > zbx-agent-1 > Encryption*: **Connections from host: PSK**, identidad `PSK-zbx-agent-1` y la clave.

### Windows (`zbx-agent-6`)

```powershell
$psk = -join ((1..64) | ForEach-Object { '{0:x}' -f (Get-Random -Maximum 16) })
Set-Content -Path "C:\Program Files\Zabbix Agent 2\zbx-agent-6.psk" -Value $psk -NoNewline
$psk
```

En `C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf`:

```
TLSConnect=psk
TLSPSKIdentity=PSK-zbx-agent-6
TLSPSKFile=C:\Program Files\Zabbix Agent 2\zbx-agent-6.psk
```

```powershell
Restart-Service "Zabbix Agent 2"
```

Y lo mismo en la pestaña *Encryption* del host `zbx-agent-6`. Los dos hosts siguen en verde y muestran el candado.

📷 CAPTURA: `capturas/26-psk-encryption.png`

📷 CAPTURA: `capturas/27-psk-hosts-verde.png`

## PASO 9: Dashboards y widgets

*Monitoring > Dashboards > Create dashboard* → `Laboratorio Zabbix - Overview`, con estos widgets:

| Widget | Configuración |
|---|---|
| Problems | Recent problems, todos los grupos, *Show tags* |
| Host availability | Todos los grupos del lab |
| Data overview | `Linux servers - Lab`, horizontal |
| Graph (classic) | CPU load de `zbx-agent-1` |
| Top hosts | `Linux servers - Lab`, columna `system.cpu.load[all,avg1]` (last) |
| Clock / System information | Opcionales |

📷 CAPTURA: `capturas/28-dashboard-overview.png`

## PASO 10: Problemas, reconocimiento y mantenimiento

Genero carga en una VM:

```bash
vagrant ssh zbx-agent-1
sudo apt-get install -y stress
stress --cpu 2 --timeout 180
```

En 1-2 minutos salta el trigger de CPU alta del template Linux en *Monitoring > Problems*. Lo **reconozco** desde la columna *Ack* con un comentario.

📷 CAPTURA: `capturas/29-problema-cpu.png`

📷 CAPTURA: `capturas/30-problema-ack.png`

Para no modificar el template oficial, si quiero otro umbral **clono** el trigger y lo ajusto.

**Mantenimiento** (*Data collection > Maintenance > Create maintenance period*): `Mantenimiento zbx-agent-1`, host `zbx-agent-1`, una vez. Dos modos:

- *With data collection*: sigue recogiendo datos pero no notifica.
- *No data collection*: deja de recoger.

📷 CAPTURA: `capturas/31-mantenimiento.png`

## PASO 11: Alertas por email (Gmail)

1. En la cuenta de Google: verificación en dos pasos activada y una **contraseña de aplicación** en `https://myaccount.google.com/apppasswords`.
2. *Alerts > Media types > Email*: SMTP `smtp.gmail.com`, puerto `587`, helo `smtp.gmail.com`, *STARTTLS*, *Username and password* con mi Gmail y la contraseña de aplicación.
3. *Users > Users > Admin > Media*: tipo Email, mi correo, `1-7,00:00-24:00`, severidades Warning o superiores.
4. *Alerts > Actions > Trigger actions > Create action* `Notificar por email - Laboratorio`: condición *Trigger severity >= Warning*, operación enviar a `Admin` solo por Email, y *Pause operations for suppressed problems* activado.

Mensaje personalizado con macros:

```
Asunto: Problema en {HOST.NAME}: {EVENT.NAME}

Severidad: {EVENT.SEVERITY}
Host: {HOST.NAME} ({HOST.IP})
Hora: {EVENT.DATE} {EVENT.TIME}
Valor actual: {ITEM.LASTVALUE}
```

Pruebo con `stress` en otra VM y recibo el correo.

📷 CAPTURA: `capturas/32-media-email.png`

📷 CAPTURA: `capturas/33-email-recibido.png`

## PASO 12: Alertas por Telegram

1. Con `@BotFather`: `/newbot`, nombre y usuario terminado en `bot`. Guardo el **token**.
2. Escribo al bot y abro `https://api.telegram.org/bot<TOKEN>/getUpdates` para sacar mi **Chat ID** (`"chat":{"id": ...}`).
3. *Alerts > Media types > Telegram* (webhook incluido en Zabbix 7.0): parámetro `Token`.
4. *Users > Users > Admin > Media*: tipo Telegram, *Send to* = Chat ID.
5. En la acción del paso anterior añado una segunda operación: enviar a `Admin` por Telegram.

Un mismo problema llega por email y por Telegram.

📷 CAPTURA: `capturas/34-telegram-recibido.png`

## PASO 13: Autorregistro con HostMetadata

El agente activo, al conectarse por primera vez, envía un **HostMetadata**; una regla del server crea el host automáticamente.

Para probarlo sin tocar los hosts existentes, añado al array `AGENTS` del `Vagrantfile` una VM `zbx-agent-test-autoreg` (IP `111`, rol `base`), la levanto con `vagrant up zbx-agent-test-autoreg` y dentro (`vagrant ssh`):

```bash
echo "HostMetadata=lab-linux-autoreg" | sudo tee -a /etc/zabbix/zabbix_agent2.conf
sudo systemctl restart zabbix-agent2
```

*Alerts > Actions > Autoregistration actions > Create action* `Autorregistro - Laboratorio Linux`:

- Condición: *Host metadata contains* `lab-linux-autoreg`.
- Operaciones: *Add host*, grupo `Linux servers - Lab`, template `Linux by Zabbix agent active`, *Enable host*.

En 1-2 minutos el host aparece solo y en verde.

Alternativa dinámica: `HostMetadataItem=system.run[cat /etc/zabbix/role-tag]` calcula el metadata a partir de un archivo.

📷 CAPTURA: `capturas/35-autorregistro.png`

## PASO 14: Autodiscovery por red

Al revés que el autorregistro: el **server** escanea la red.

*Data collection > Discovery > Create discovery rule* `Descubrimiento LAN - Laboratorio`:

- IP range `192.168.1.100-192.168.1.110`, intervalo `1h`.
- Checks: *Zabbix agent* (puerto `10050`, key `system.uname`), *ICMP ping* y opcionalmente *SNMPv2* (`161`).

*Alerts > Actions > Discovery actions*: condición *Service type = ICMP ping* y *Discovery rule = Descubrimiento LAN*, operación *Add host* al grupo `Descubiertos - Lab`.

Los resultados se ven en *Monitoring > Discovery*. Los hosts ya creados no se duplican.

📷 CAPTURA: `capturas/36-discovery.png`

## PASO 15: Monitorización SNMP (`zbx-agent-10`)

`provision/snmp-proxy.sh` ya instaló `snmpd` con una configuración mínima (comunidad `public` de solo lectura):

```
rocommunity public default
syslocation "Laboratorio Zabbix"
syscontact  admin@lab.local
```

Pruebo desde otra VM:

```bash
sudo apt install -y snmp
snmpwalk -v2c -c public 192.168.1.110 system
```

📷 CAPTURA: `capturas/37-snmpwalk.png`

Creo un host **separado** `zbx-agent-10-snmp` (grupo `Network devices - Lab`), con interfaz **SNMP** `192.168.1.110:161`, SNMPv2, comunidad `public`, y el template `Generic by SNMP`.

> [!warning] Por qué un host aparte
> Si añado la interfaz SNMP al propio `zbx-agent-10`, al guardar aparece *Cannot inherit item with key "system.name"… inventory field "Name" is already populated*: el template Linux y el SNMP quieren escribir el mismo campo del inventario. Con dos hosts no hay conflicto (la otra opción es poner *Populates host inventory field* a *None* en el item `system.name` del template SNMP).

📷 CAPTURA: `capturas/38-host-snmp.png`

## PASO 16: MySQL con el plugin de Agent 2 (`zbx-agent-3`)

Usuario de monitorización con permisos mínimos:

```bash
sudo mysql
```

```sql
CREATE USER 'zbx_monitor'@'localhost' IDENTIFIED BY 'MonitorPass123!';
GRANT REPLICATION CLIENT, PROCESS, SHOW DATABASES ON *.* TO 'zbx_monitor'@'localhost';
GRANT SELECT ON performance_schema.* TO 'zbx_monitor'@'localhost';
FLUSH PRIVILEGES;
```

`/etc/zabbix/zabbix_agent2.d/plugins.d/mysql.conf`:

```
Plugins.Mysql.Default.User=zbx_monitor
Plugins.Mysql.Default.Password=MonitorPass123!
```

> Es `Plugins.Mysql.Default.*`, **no** `Plugins.Mysql.Sessions.Default.*`: las *Sessions* solo se usan cuando el item recibe el nombre de sesión como parámetro.

```bash
sudo systemctl restart zabbix-agent2
sudo zabbix_agent2 -t mysql.ping       # 1
sudo zabbix_agent2 -t mysql.version
```

> `zabbix_agent2 -t` siempre con `sudo`: sin él falla con *bind: permission denied* en `/run/zabbix/`.

Template **`MySQL by Zabbix agent 2 active`** (la variante pasiva da *a host interface of type Agent is required*). Tras **Add** hay que pulsar **Update** al final del formulario, si no, no se guarda. Macros del host:

| Macro | Valor |
|---|---|
| `{$MYSQL.HOST}` | `localhost` |
| `{$MYSQL.PORT}` | `3306` |
| `{$MYSQL.USER}` | `zbx_monitor` |
| `{$MYSQL.PASSWORD}` | `MonitorPass123!` |

Carga de prueba:

```bash
sudo mysql -e "CREATE DATABASE IF NOT EXISTS testdb; CREATE TABLE IF NOT EXISTS testdb.t (id INT);"
for i in $(seq 1 1000); do sudo mysql -e "INSERT INTO testdb.t VALUES ($i);"; done
```

📷 CAPTURA: `capturas/39-mysql-agent-test.png`

📷 CAPTURA: `capturas/40-mysql-latest-data.png`

## PASO 17: PostgreSQL (`zbx-agent-7`)

```bash
sudo -u postgres psql
```

```sql
CREATE USER zbx_monitor WITH PASSWORD 'MonitorPass123!';
GRANT pg_monitor TO zbx_monitor;
\q
```

En `/etc/postgresql/16/main/pg_hba.conf`:

```
host    all    zbx_monitor    127.0.0.1/32    scram-sha-256
```

`/etc/zabbix/zabbix_agent2.d/plugins.d/postgresql.conf`:

```
Plugins.PostgreSQL.Default.User=zbx_monitor
Plugins.PostgreSQL.Default.Password=MonitorPass123!
Plugins.PostgreSQL.Default.Uri=tcp://localhost:5432
```

```bash
sudo systemctl restart postgresql zabbix-agent2
sudo zabbix_agent2 -t pgsql.ping       # 1
```

Template `PostgreSQL by Zabbix agent 2` (o su variante *active* si da el error de interfaz). Macros `{$PG.HOST}`=`localhost`, `{$PG.PORT}`=`5432`, `{$PG.USER}`=`zbx_monitor`, `{$PG.PASSWORD}`=`MonitorPass123!`, `{$PG.URI}`=`tcp://localhost:5432`.

Carga con `pgbench`:

```bash
sudo -u postgres createdb testdb
sudo -u postgres pgbench -i testdb
sudo -u postgres pgbench -c 5 -T 30 testdb
```

📷 CAPTURA: `capturas/41-pgsql-latest-data.png`

## PASO 18: Redis (`zbx-agent-8`)

`provision/redis.sh` ya dejó Redis con `requirepass MonitorPass123!` y `enable-debug-command local`.

`/etc/zabbix/zabbix_agent2.d/plugins.d/redis.conf`:

```
Plugins.Redis.Default.Uri=tcp://localhost:6379
Plugins.Redis.Default.Password=MonitorPass123!
```

```bash
sudo systemctl restart zabbix-agent2
sudo zabbix_agent2 -t redis.ping
sudo zabbix_agent2 -t redis.info       # muestra el error real si falla (p. ej. NOAUTH)
```

Template `Redis by Zabbix agent 2`, macros `{$REDIS.CONN.URI}`=`tcp://localhost:6379` y `{$REDIS.PASSWORD}`=`MonitorPass123!`.

Carga de prueba:

```bash
redis-cli -a 'MonitorPass123!' --no-auth-warning DEBUG POPULATE 10000
```

Trigger propio (*Hosts > zbx-agent-8 > Triggers*): `Redis usando más de 100MB en {HOST.NAME}`, expresión `last(/zbx-agent-8/redis.info[used_memory,tcp://localhost:6379,Default]) > 104857600` (compruebo antes la key exacta en *Latest data*).

📷 CAPTURA: `capturas/42-redis-latest-data.png`

## PASO 19: Nginx con UserParameters (`zbx-agent-2`)

Nginx no tiene plugin nativo; expone sus métricas con `stub_status` y las leo con **UserParameters**.

`/etc/nginx/conf.d/status.conf`:

```nginx
server {
    listen 127.0.0.1:8081;
    location /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
curl http://127.0.0.1:8081/nginx_status
```

`/etc/zabbix/zabbix_agent2.d/userparameter_nginx.conf`:

```
UserParameter=nginx.active_connections,curl -s http://127.0.0.1:8081/nginx_status | awk '/Active/ {print $3}'
UserParameter=nginx.requests_total,curl -s http://127.0.0.1:8081/nginx_status | awk 'NR==3 {print $3}'
UserParameter=nginx.reading,curl -s http://127.0.0.1:8081/nginx_status | awk '/Reading/ {print $2}'
UserParameter=nginx.writing,curl -s http://127.0.0.1:8081/nginx_status | awk '/Writing/ {print $4}'
UserParameter=nginx.waiting,curl -s http://127.0.0.1:8081/nginx_status | awk '/Waiting/ {print $6}'
```

```bash
sudo systemctl restart zabbix-agent2
sudo zabbix_agent2 -t nginx.active_connections
```

Creo los 5 items en el host (tipo *Zabbix agent (active)*, *Numeric (unsigned)*, `1m`) y un trigger `last(/zbx-agent-2/nginx.active_connections) > 100`. Carga:

```bash
sudo apt install -y apache2-utils
ab -n 5000 -c 20 http://127.0.0.1/
```

📷 CAPTURA: `capturas/43-nginx-status.png`

📷 CAPTURA: `capturas/44-nginx-latest-data.png`

## PASO 20: Contenedores Docker (`zbx-agent-9`)

`provision/docker.sh` ya instaló Docker y metió al usuario `zabbix` en el grupo `docker` (necesario para leer `/var/run/docker.sock`).

```bash
sudo docker run -d --name web-test -p 8888:80 nginx
sudo docker run -d --name db-test -e MYSQL_ROOT_PASSWORD=test mysql:8.0
sudo zabbix_agent2 -t 'docker.containers[all]'
```

Template `Docker by Zabbix agent 2`: usa **discovery LLD** y crea items y triggers por cada contenedor.

Pruebo un contenedor caído:

```bash
sudo docker stop web-test
```

y aparece el problema en *Monitoring > Problems*.

📷 CAPTURA: `capturas/45-docker-lld.png`

📷 CAPTURA: `capturas/46-docker-problema.png`

## PASO 21: RabbitMQ sin agente, con HTTP Agent (`zbx-agent-4`)

Aquí es el **server** el que consulta la API HTTP de RabbitMQ. `provision/rabbitmq.sh` ya instaló RabbitMQ con el plugin de management y creó el usuario `zbx_monitor` (tag `monitoring`, sin permisos sobre las colas). Compruebo la API:

```bash
curl -u 'zbx_monitor:MonitorPass123!' http://localhost:15672/api/overview
```

Item principal en `zbx-agent-4`:

- **Type**: HTTP agent, **Key**: `rabbitmq.overview.raw`.
- **URL**: `http://192.168.1.104:15672/api/overview`, autenticación *Basic* con `zbx_monitor`.
- **Type of information**: Text, `1m`.

Item **dependiente** que extrae un valor del JSON:

- **Type**: Dependent item, master `RabbitMQ: Overview (raw)`, key `rabbitmq.queue_totals.messages`, *Numeric (unsigned)*.
- **Preprocessing**: JSONPath `$.queue_totals.messages`.

Tráfico de prueba:

```bash
sudo apt install -y python3-pika
python3 - <<'PY'
import pika
conn = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
ch = conn.channel()
ch.queue_declare(queue='cola_prueba')
for i in range(500):
    ch.basic_publish(exchange='', routing_key='cola_prueba', body=f'mensaje {i}')
conn.close()
PY
```

Trigger: `last(/zbx-agent-4/rabbitmq.queue_totals.messages) > 1000`.

📷 CAPTURA: `capturas/47-rabbitmq-api.png`

📷 CAPTURA: `capturas/48-rabbitmq-items.png`

## PASO 22: Mapa de red

*Monitoring > Maps > Create map* `Laboratorio Zabbix - Topología` (1200x800):

- Un elemento *Host* por cada host, con `Zabbix server` en el centro.
- Enlaces de cada agente al server (Ctrl+clic en dos elementos > *Add link*), con *Link indicators* que ponen el enlace en rojo si el host tiene problemas.
- Organizo los iconos por rol (Linux, bases de datos, mensajería/otros).

Lo añado como widget *Map* al dashboard del PASO 9. Al apagar una VM (`vagrant halt zbx-agent-3`) su icono cambia de color en 1-2 minutos.

📷 CAPTURA: `capturas/49-mapa-red.png`

## PASO 23: Template propio desde cero

*Data collection > Templates > Create template* `Custom Nginx Monitoring`, grupo `Custom templates - Lab`:

- **Items**: los 5 de Nginx del PASO 19, con el tag `component: nginx`.
- **Item calculado** `nginx.reading.percent`: `last(//nginx.reading) / last(//nginx.active_connections) * 100` (`//` = el mismo host donde se aplica).
- **Triggers**:
  - Warning: `last(/Custom Nginx Monitoring/nginx.active_connections) > 100`
  - High: `nodata(/Custom Nginx Monitoring/nginx.active_connections,5m) = 1` (el servicio deja de responder)
- **Gráfico** `Nginx - Conexiones` con active, reading y writing.

Lo aplico a `zbx-agent-2` (Add + **Update**), borro los items sueltos del PASO 19 para no duplicar y lo **exporto en YAML** (*Templates > Export*).

📷 CAPTURA: `capturas/50-template-propio.png`

## PASO 24: Zabbix Proxy, sede remota (`zbx-agent-10`)

`provision/snmp-proxy.sh` ya instaló `zabbix-proxy-sqlite3` (SQLite, para no necesitar otra base de datos) con:

```
Server=192.168.1.100
Hostname=zbx-proxy-remoto
DBName=/var/lib/zabbix/zabbix_proxy.db
```

> Guardo la base en `/var/lib/zabbix` y no en `/tmp`, que se vacía al reiniciar.

```bash
vagrant ssh zbx-agent-10
systemctl status zabbix-proxy --no-pager
```

*Administration > Proxies > Create proxy*: nombre `zbx-proxy-remoto` (igual que el `Hostname` del `.conf`), modo **Active**.

En *Hosts > zbx-agent-10* pongo **Monitored by proxy** = `zbx-proxy-remoto`. Para que el agente hable con el proxy, cambio en su `.conf`:

```
Server=127.0.0.1,192.168.1.100
ServerActive=127.0.0.1
```

```bash
sudo systemctl restart zabbix-agent2
sudo tail -f /var/log/zabbix/zabbix_proxy.log
```

Simulo un corte de enlace con `sudo systemctl stop zabbix-proxy` y al arrancarlo de nuevo los datos pendientes se envían sin perder nada.

📷 CAPTURA: `capturas/51-proxy-frontend.png`

📷 CAPTURA: `capturas/52-proxy-log.png`

## PASO 25: Backup y restauración

Toda la configuración e historial están en la base del contenedor `zabbix-mysql`.

```powershell
# Backup completo
docker exec zabbix-mysql sh -c 'exec mysqldump -uroot -p"root_pwd" zabbix' > zabbix_backup_$(Get-Date -Format "yyyyMMdd_HHmmss").sql

# Solo configuración (sin tablas de historial ni tendencias)
docker exec zabbix-mysql mysqldump -uroot -p"root_pwd" `
  --ignore-table=zabbix.history --ignore-table=zabbix.history_uint `
  --ignore-table=zabbix.history_str --ignore-table=zabbix.history_text `
  --ignore-table=zabbix.history_log --ignore-table=zabbix.trends `
  --ignore-table=zabbix.trends_uint zabbix > zabbix_config_only.sql
```

Restaurar (parando antes el server):

```powershell
docker compose stop zabbix-server
Get-Content zabbix_backup_XXXX.sql | docker exec -i zabbix-mysql mysql -uroot -p"root_pwd" zabbix
docker compose start zabbix-server
```

Lo probé de verdad: backup → borro un dashboard → restauro → el dashboard vuelve. Además programé un `backup-zabbix.ps1` diario en el **Programador de tareas** que guarda los 7 últimos.

📷 CAPTURA: `capturas/53-backup.png`

📷 CAPTURA: `capturas/54-restauracion.png`

## PASO 26: Integración con Grafana

Añado al `docker-compose.yml`:

```yaml
  grafana:
    image: grafana/grafana-oss:latest
    container_name: zabbix-grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      GF_INSTALL_PLUGINS: alexanderzobnin-zabbix-app
    networks:
      - zbx-net
    volumes:
      - grafana-data:/var/lib/grafana
```

y `grafana-data:` en `volumes:`.

```powershell
docker compose up -d grafana
```

En `http://localhost:3000` (`admin`/`admin`, pide cambiarla): *Administration > Plugins > Zabbix* → Enable. *Connections > Data sources > Zabbix*: URL `http://zabbix-web:8080/api_jsonrpc.php` (nombre del contenedor, misma red `zbx-net`) y el usuario `Admin` de Zabbix → *Save & test*.

Dashboard con paneles de varios hosts y una variable `host` para cambiar de host con un desplegable.

📷 CAPTURA: `capturas/55-grafana-datasource.png`

📷 CAPTURA: `capturas/56-grafana-dashboard.png`

## PASO 27: Actualización de versión

- **Menor** (7.0.x → 7.0.y): con el tag `ubuntu-7.0-latest` basta con `docker compose pull` y `docker compose up -d`.
- **Mayor** (7.0 → siguiente LTS): solo entre versiones mayores consecutivas y siempre con backup:

```powershell
docker exec zabbix-mysql sh -c 'exec mysqldump -uroot -p"root_pwd" zabbix' > zabbix_pre_upgrade_backup.sql
docker compose down
# cambiar ubuntu-7.0-latest por la nueva versión en server y web
docker compose up -d
docker compose logs zabbix-server -f     # migración de esquema automática
```

**Rollback**: `docker compose down`, volver a los tags anteriores, `up -d` y restaurar el backup.

Los agentes se actualizan a la misma versión mayor: en las VM Linux cambio la URL del paquete `zabbix-release` en `common-agent.sh` y ejecuto `vagrant provision <host>`; en Windows reinstalo el MSI nuevo.

## PASO 28: Housekeeper y mantenimiento

*Administration > Housekeeping*:

- Trigger data: `1 year`.
- History: *Override item history period* `31d`.
- Trends: *Override item trend period* `365d`.

Se puede ajustar la retención por item (*History storage period*). El host `Zabbix server` sirve para vigilar al propio server (cola, procesos internos, housekeeper).

| Frecuencia | Tarea |
|---|---|
| Diaria | Backup de configuración automático |
| Semanal | *Reports > Availability report* |
| Mensual | Tamaño del volumen `mysql-data` y retención |
| Cada actualización mayor | Backup completo + procedimiento del PASO 27 |

📷 CAPTURA: `capturas/57-housekeeping.png`

## Referencia rápida

### Parámetros del agente

| Parámetro | Uso |
|---|---|
| `Server` | IPs que pueden hacer consultas pasivas |
| `ServerActive` | Dónde envía los datos en modo activo |
| `Hostname` | Nombre exacto del host en el frontend |
| `RefreshActiveChecks` | Cada cuánto pide su lista de checks (60 s) |
| `Timeout` | Tiempo máximo por check |
| `TLSConnect` / `TLSAccept` / `TLSPSKIdentity` / `TLSPSKFile` | Cifrado PSK |
| `HostMetadata` / `HostMetadataItem` | Autorregistro |
| `UserParameter=<clave>,<comando>` | Métricas propias |
| `LogFile` / `DebugLevel` | Log y nivel de detalle |

```bash
sudo zabbix_agent2 -t <key>                # probar un item
sudo zabbix_agent2 -p                      # listar items soportados
journalctl -u zabbix-agent2 -f
tail -f /var/log/zabbix/zabbix_agent2.log
```

### Puertos

| Puerto | Servicio | Sentido |
|---|---|---|
| `8080` | Frontend | Navegador → `.100` |
| `10051` | Zabbix Server (activo) | Agentes / proxy → `.100` |
| `10050` | Agente (pasivo) | Referencia de interfaz |
| `161` | SNMP | Server → `.110` |
| `15672` | API de RabbitMQ | Server → `.104` |
| `389` | LDAP (AD) | Frontend → `.106` |
| `3000` | Grafana | Navegador → `.100` |

## Problemas encontrados

| Problema | Causa | Solución |
|---|---|---|
| `vagrant up` falla con *linux-headers… has no installation candidate* | El plugin `vagrant-vbguest` intenta compilar las Guest Additions | `config.vbguest.auto_update = false` en el `Vagrantfile` |
| `vagrant up` se para preguntando por un adaptador | Red puente interactiva la primera vez | Elegir el adaptador Wi-Fi/Ethernet con salida a internet |
| El provisioning falla descargando paquetes | La VM no tiene salida a internet | `vagrant ssh <host>` y `ping -c3 8.8.8.8`; revisar el adaptador puente |
| `zabbix-agent2-plugin-mysql` no existe | El plugin de MySQL viene dentro de `zabbix-agent2` | No instalarlo aparte |
| Los agentes no conectan con el server | Firewall de Windows bloqueando el `10051` | Regla de entrada del PASO 2 |
| Host en rojo, "host not found" | `Host name` del frontend distinto del `Hostname` del agente | Poner exactamente el mismo nombre |
| Host gris sin datos | Template pasivo (`Linux by Zabbix agent`) | Usar `Linux by Zabbix agent active` |
| El template "no se aplica" | Se pulsó *Add* pero no *Update* | Pulsar **Update** al final del formulario |
| *a host interface of type Agent is required* | Template pasivo de servicio | Usar la variante *active* |
| `zabbix_agent2 -t` → *bind: permission denied* | Ejecutado sin `sudo` | Usar `sudo zabbix_agent2 -t` |
| Redis → `NOAUTH Authentication required` | Contraseña en `Sessions.Default.*` | Usar `Plugins.Redis.Default.*` |
| *Cannot inherit item "system.name"* al añadir SNMP | Dos templates escriben el mismo campo de inventario | Host SNMP separado |
| Host tras PSK en rojo | `TLSPSKIdentity` distinta o permisos del `.psk` | Revisar identidad y que `zabbix` pueda leer el archivo |
| Contraseñas en texto plano | Simplicidad del lab | En real, `.env` o secretos de Docker |

📷 CAPTURA (opcional): `capturas/problema-*.png` para cada error que quieras mostrar

## Resultado final

- [ ] Zabbix Server 7.0 en Docker, reproducible con `docker compose up -d`.
- [ ] 8 VM Linux levantadas con un solo `Vagrantfile` y en verde.
- [ ] Solaris con agente clásico compilado y Windows Server (Vagrant) con AD + LDAP.
- [ ] PSK en dos hosts.
- [ ] Dashboard, gestión de problemas y alertas por email y Telegram.
- [ ] Autorregistro y autodiscovery.
- [ ] SNMP + 6 servicios con 3 métodos (plugin nativo, UserParameters, HTTP Agent).
- [ ] Mapa, template propio exportado, proxy, backups probados, Grafana y housekeeping.

📷 CAPTURA: `capturas/58-dashboard-final.png` (dashboard final con todos los hosts)
