# Solución

## Arquitectura

Para este laboratorio he montado **dos VM** en VirtualBox, las dos con Ubuntu 24.04 LTS y conectadas a la red local en modo puente:

| VM              | Qué tendrá instalado                                               | IP             |
| --------------- | ------------------------------------------------------------------ | -------------- |
| `vm-docker-app` | Linux Server + Docker + la app en Python (dentro de un contenedor) | `192.168.1.60` |
| `vm-postgres`   | Linux Server + PostgreSQL 18 como servicio nativo                  | `192.168.1.61` |

![](capturas/Pasted%20image%2020260923093221.png)

En cada paso indico en qué VM se ejecuta cada comando.

## PASO 1: Configuraciones generales

### Instalar Ubuntu 24 y preparar el entorno

![](capturas/Pasted%20image%2020260921234441.png)

Creé las dos VM con Ubuntu 24.04 LTS. El usuario de trabajo no root, `ciaoyaniz`, lo creé durante la propia instalación, y así evito trabajar con `root`.

Durante la instalación también activé el servidor **OpenSSH** para poder conectarme a las VM de forma remota desde mi PC.

> **Red**: en modo puente (Bridged), cada VM se comporta como una máquina más de la red local (`192.168.1.0/24`) y tiene su propia IP dentro de la red. Así las dos VM se ven entre ellas y también desde mi PC.

### Configurar IP fija en cada VM

Por defecto cada VM recibe su IP por DHCP del router, y esa IP puede cambiar al reiniciar. Como la app y `pg_hba.conf` dependen de la IP de cada VM, les asigné una **IP fija** fuera del rango habitual del DHCP, a partir de la `.60`:

| VM | IP fija |
|---|---|
| `vm-docker-app` | `192.168.1.60` |
| `vm-postgres` | `192.168.1.61` |

> Nota: en las primeras capturas `vm-docker-app` aparece con la IP `192.168.1.44`, que era la que le había dado el DHCP antes de fijarla.

Primero consulto el nombre de la interfaz de red, la puerta de enlace y el archivo de configuración de red. Ubuntu Server gestiona la red con **Netplan**, que usa archivos YAML en `/etc/netplan/`:

```bash
ip a
ip route | grep default
ls /etc/netplan/
```

![](capturas/Pasted%20image%2020260922214103.png)

En mi caso la interfaz es `enp0s3` y la puerta de enlace (el router) es `192.168.1.1`.

Edito el archivo de Netplan (el nombre puede variar, por ejemplo `00-cloud-init.yaml`):

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Contenido en `vm-docker-app`:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.1.60/24     # Modifico esta línea para cada VM
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1, 8.8.8.8]
```

En `vm-postgres` es igual, cambiando la dirección por `192.168.1.61/24`.

- `dhcp4: false`: desactiva la IP automática.
- `addresses`: la IP fija con su máscara (`/24` = `255.255.255.0`).
- `routes`: la puerta de enlace por defecto para salir a internet.
- `nameservers`: servidores DNS (el router y el de Google como respaldo).

![](capturas/Pasted%20image%2020260922220112.png)

![](capturas/Pasted%20image%2020260922220157.png)

> [!TIP] Opcional: Ajustar configuraciones de cloud-init
> En muchos servidores (Ubuntu en la nube, máquinas virtuales, etc.), **cloud-init** configura automáticamente la red cada vez que arranca el sistema.
> 
> Si el archivo lo genera `cloud-init`, hay que desactivar su gestión de red para que no sobrescriba mis cambios al reiniciar:
> 
> ```bash
> echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
> ```
> 
> Ajusto los permisos del archivo (solo `root`puede leer y escribir) y aplico la configuración con `netplan try`, que la revierte sola a los 120 segundos si no confirmo con `Enter`. Así no pierdo el acceso si me equivoco:
> 
> ```bash
> sudo chmod 600 /etc/netplan/50-cloud-init.yaml
> sudo netplan try
> ```
>
>Netplan suele mostrar advertencias si sus archivos son legibles por otros usuarios porque pueden contener información sensible de red.

> [!warning] Al aplicar la IP nueva se corta la conexión SSH
> La sesión SSH estaba abierta con la IP antigua, así que se queda colgada. Para confirmar el cambio, lo hice desde la consola de VirtualBox, o volví a conectarme con la IP nueva.

Compruebo la IP nueva, la salida a internet y que las dos VM se ven entre ellas:

```bash
ip a show enp0s3
ping -c 3 8.8.8.8
ping -c 3 192.168.1.61   # ejecuto desde desde vm-docker-app
```

![](capturas/Pasted%20image%2020260922220702.png)

### Actualización del sistema

Una vez dentro de cada VM, actualicé el sistema y reinicié para cargar el kernel nuevo:

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

- `apt update`: descarga la lista actualizada de paquetes disponibles.
- `apt upgrade -y`: instala las versiones nuevas de los paquetes ya instalados (`-y` responde "sí" a todo).

![](capturas/Pasted%20image%2020260921223035.png)

![](capturas/Pasted%20image%2020260921223051.png)

### Asignar permisos `sudo` a mi usuario

`usermod -aG sudo` agrega mi usuario al grupo `sudo` y me permite ejecutar comandos administrativos con `sudo` sin iniciar sesión como `root`. La opción `-a` (append) es importante: sin ella, `-G` reemplazaría todos los grupos del usuario en vez de añadir uno.

```bash
sudo usermod -aG sudo ciaoyaniz
groups ciaoyaniz
```

> Nota: en Ubuntu el primer usuario creado en la instalación ya pertenece al grupo `sudo`, así que este paso sirve sobre todo para comprobarlo. Con `groups` veo a qué grupos pertenece el usuario.

![](capturas/Pasted%20image%2020260922221018.png)

Repetí el proceso en las dos VM.

## PASO 2: Instalar Docker (en `vm-docker-app`)

### Instalar Docker

Para instalar Docker preparé un script con ayuda de IA. Antes de ejecutarlo lo revisé para entender qué hace cada bloque:

1. Borra configuraciones anteriores del repositorio de Docker, por si quedaba algo de un intento previo.
2. Actualiza el sistema.
3. Instala las dependencias necesarias (`curl`, `gnupg`, certificados…).
4. Descarga la **clave GPG** oficial de Docker, que sirve para que `apt` compruebe que los paquetes vienen realmente de Docker, y agrega el repositorio oficial.
5. Instala Docker Engine, el cliente, `containerd` y los plugins de `buildx` y `compose`.
6. Agrega mi usuario al grupo `docker` y deja el servicio arrancado y habilitado al inicio.

Para crearlo, darle permisos de ejecución y ejecutarlo:

```bash
nano install-docker.sh
chmod +x install-docker.sh
./install-docker.sh
```

```bash
#!/bin/bash

set -e

echo "========================================"
echo "Limpiando e instalando Docker"
echo "========================================"

# 1. Limpiar configuraciones anteriores
echo "[1/5] Limpiando configuraciones anteriores..."
sudo rm -f /etc/apt/sources.list.d/docker.list
sudo rm -f /etc/apt/keyrings/docker.asc
echo "✓ Configuraciones previas eliminadas"

# 2. Actualizar repositorios ANTES de agregar Docker
echo "[2/5] Actualizando repositorios..."
sudo apt clean
sudo apt update
sudo apt upgrade -y

# 3. Instalar dependencias
echo "[3/5] Instalando dependencias..."
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https

# 4. Crear directorio y descargar clave GPG
echo "[4/5] Agregando repositorio de Docker..."
sudo mkdir -p /etc/apt/keyrings

# Descargar la clave GPG
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Agregar repositorio
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Actualizar repositorios CON la clave ya instalada
sudo apt update

# 5. Instalar Docker
echo "[5/5] Instalando Docker..."
sudo apt install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin

# Verificar instalación
echo ""
echo "========================================"
echo "✓ Docker instalado exitosamente"
echo "========================================"
docker --version
docker compose version

# Configurar usuario
echo ""
echo "⚙️  Configurando permisos..."
sudo usermod -aG docker $USER

# Iniciar servicio
sudo systemctl start docker
sudo systemctl enable docker

echo ""
echo "✅ ¡Instalación completada!"
echo ""
echo "IMPORTANTE: Debes cerrar sesión y volver a conectarte"
echo "Ejecuta: exit"
echo "Luego reconéctate y prueba: docker run hello-world"
```

La instalación tardó un poco, pero el script funcionó correctamente:

![](capturas/Pasted%20image%2020260922124241.png)

### Grupo `docker`

El paquete `docker-ce` crea el grupo `docker` automáticamente, pero lo compruebo y lo creo si no existe. Después agrego mi usuario para poder usar Docker sin `sudo`:

```bash
getent group docker || sudo groupadd docker
sudo usermod -aG docker $USER
```

Los grupos nuevos no se aplican a la sesión abierta, así que cerré la sesión SSH con `exit` y volví a conectarme. También se puede aplicar en la sesión actual con `newgrp docker`.

![](capturas/Pasted%20image%2020260922221037.png)

### Validar la instalación

Levanté el contenedor de prueba `hello-world` **sin `sudo`**, lo que confirma que Docker funciona y que los permisos del grupo están bien:

```bash
docker run hello-world
```

Docker no encuentra la imagen en local, la descarga de Docker Hub, crea un contenedor, lo ejecuta y muestra el mensaje de bienvenida:

![](capturas/Pasted%20image%2020260922001758.png)

## PASO 3: Instalar PostgreSQL 18 (en `vm-postgres`)

### Instalar PostgreSQL sobre Ubuntu

Los repositorios de Ubuntu 24.04 traen PostgreSQL 16, así que para instalar la versión 18 primero agregué el repositorio oficial de PostgreSQL (PGDG) con su clave y actualicé la lista de paquetes:

```bash
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
  --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
> /etc/apt/sources.list.d/pgdg.list'

sudo apt update
```

Instalo el servidor y el cliente de PostgreSQL 18:

```bash
sudo apt install -y postgresql-18 postgresql-client-18
```

### Verificar que está corriendo como servicio

La instalación registra PostgreSQL como servicio de `systemd`, así que arranca solo cada vez que se inicia la VM. Lo compruebo con:

```bash
sudo systemctl status postgresql
```

![](capturas/Pasted%20image%2020260922125646.png)

> Nota: que aparezca `active (exited)` es normal. `postgresql.service` es solo un servicio "paraguas" que arranca los clusters; el proceso real del servidor es `postgresql@18-main`, que se puede ver con `sudo systemctl status postgresql@18-main`.

En Ubuntu, PostgreSQL organiza sus instancias en **clusters**: cada cluster es una instancia independiente, con su versión, su puerto y su carpeta de datos. Esto permite tener varias versiones instaladas a la vez. Los listo con:

```bash
pg_lsclusters
```

![](capturas/Pasted%20image%2020260922130114.png)

El resultado muestra un único cluster `main` con PostgreSQL 18, en estado `online` y escuchando en el puerto `5432`.

### Ingresar al prompt por terminal

Durante la instalación PostgreSQL crea un usuario del sistema y de la base de datos llamado `postgres`, que es el administrador. Para entrar a la consola `psql` lo hago con ese usuario y compruebo la versión:

```bash
sudo -u postgres psql
```

```sql
SELECT version();
```

![](capturas/Pasted%20image%2020260922130940.png)

## PASO 4: Crear base de datos, usuario y tabla (en `vm-postgres`)

### Crear la base de datos

Dentro de `psql` creo la base de datos que va a usar mi app en Python y la compruebo con `\l`, que lista las bases de datos:

```sql
CREATE DATABASE labdb;
\l
```

![](capturas/Pasted%20image%2020260922141646.png)

### Crear el usuario de la aplicación

No quiero que la app se conecte con `postgres`, que es el administrador, así que creo un usuario propio para la aplicación, con los permisos justos para trabajar sobre `labdb`:

```sql
CREATE USER labapp WITH PASSWORD 'labapp_26';
GRANT ALL PRIVILEGES ON DATABASE labdb TO labapp;
```

Los permisos de PostgreSQL funcionan por niveles: base de datos → esquema → tablas. El `GRANT` anterior solo da permisos a nivel de base de datos (conectarse, crear esquemas), así que también hay que darle permisos sobre el esquema `public`. Para eso me conecto a `labdb`:

```sql
\c labdb
GRANT ALL ON SCHEMA public TO labapp;
```

> Nota: un **esquema** es un espacio de nombres dentro de una base de datos donde se guardan las tablas, vistas, secuencias, etc. `public` es el esquema por defecto. Desde PostgreSQL 15, por seguridad, los usuarios normales ya no pueden crear objetos en `public` si no se les da permiso explícitamente.

### Crear una tabla de ejemplo

```sql
CREATE TABLE productos (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    precio NUMERIC(10, 2) NOT NULL,
    creado_en TIMESTAMP DEFAULT NOW()
);
```

- `SERIAL PRIMARY KEY`: id numérico que se autoincrementa (usa una secuencia por detrás).
- `NUMERIC(10, 2)`: número con 2 decimales, adecuado para precios.
- `DEFAULT NOW()`: guarda automáticamente la fecha de creación de cada fila.

Inserto algunos datos de prueba:

```sql
INSERT INTO productos (nombre, precio) VALUES
    ('Teclado mecánico', 45.99),
    ('Mouse inalámbrico', 19.50),
    ('Monitor 24 pulgadas', 149.00),
    ('Webcam HD', 32.75);
```

![](capturas/Pasted%20image%2020260922150315.png)

### Dar permisos sobre la tabla

Como la tabla la creé con el usuario `postgres`, su dueño es `postgres` y `labapp` todavía no puede leerla. Le doy permisos sobre las tablas y secuencias del esquema **después** de haber creado la tabla:

```sql
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO labapp;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO labapp;
```

> [!warning] Cuidado con el orden de los permisos
> `GRANT ... ON ALL TABLES` solo afecta a las tablas que **ya existen** en el momento de ejecutarlo. Si se ejecuta antes del `CREATE TABLE`, `labapp` no tendrá permisos sobre la tabla nueva y la app fallará con `permission denied for table productos`. Para que las tablas futuras también hereden los permisos se puede usar:
> ```sql
> ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO labapp;
> ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO labapp;
> ```

Compruebo los permisos de la tabla con `\dp`:

```sql
\dp productos
```

![](capturas/Pasted%20image%2020260922223714.png)

Esto quiere decir que `labapp` tiene todos los permisos y se los dio `postgres`. Salgo de `psql` con `\q`.

### Comprobar el puerto de PostgreSQL

Compruebo que PostgreSQL está escuchando en el puerto 5432:

```bash
sudo ss -tlnp | grep 5432
```

![](capturas/Pasted%20image%2020260922202654.png)

Por defecto, PostgreSQL recién instalado solo escucha en `127.0.0.1` (`localhost`), así que de momento solo acepta conexiones desde la propia `vm-postgres`. Lo cambio más adelante, cuando conecte la app.

## PASO 5: Crear una aplicación en Python (en `vm-docker-app`)

Antes de meter la app en Docker, la pruebo directamente sobre la VM. Así, si algo falla, sé si el problema está en el código o en Docker.

### Instalar Python

Ubuntu 24.04 ya trae Python 3, pero hay que instalar también `pip` (gestor de paquetes) y `venv` (entornos virtuales):

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv

python3 --version
pip3 --version
```

### Crear un entorno virtual para el proyecto

Creo la carpeta de trabajo:

```bash
mkdir -p ~/lab-postgres-docker
cd ~/lab-postgres-docker
```

Creo y activo un **entorno virtual**. Sirve para aislar las dependencias de este proyecto del resto del sistema: todo lo que instale con `pip` queda dentro de la carpeta `venv`:

```bash
python3 -m venv venv
source venv/bin/activate
```

Al activarlo, el prompt muestra `(venv)` al principio:

![](capturas/Pasted%20image%2020260922193428.png)

### Instalar las dependencias

Instalo las librerías que necesita la app y guardo la lista exacta en `requirements.txt`, que después usaré para instalar lo mismo dentro de Docker:

```bash
pip install flask psycopg2-binary python-dotenv
pip freeze > requirements.txt
```

- `psycopg2-binary`: el driver para conectarse a PostgreSQL desde Python.
- `flask`: framework web mínimo para exponer el resultado en un endpoint HTTP.
- `python-dotenv`: carga las variables del archivo `.env`, así no escribo las credenciales dentro del código.

### Crear archivo con variables de entorno

Creo el archivo `.env` en la raíz del proyecto con `nano .env`:

```
DB_HOST=localhost
DB_PORT=5432
DB_NAME=labdb
DB_USER=labapp
DB_PASSWORD=labapp_26
PORT=3000
```

![](capturas/Pasted%20image%2020260922201025.png)

> Nota: este archivo contiene la contraseña de la base de datos, por eso queda excluido del repositorio Git y de la imagen de Docker.

### Crear la app

Creo `app.py` en la raíz del proyecto con `nano app.py`:

```python
import os
import sys

from dotenv import load_dotenv
from flask import Flask, jsonify
import psycopg2
import psycopg2.extras

load_dotenv()

app = Flask(__name__)

DB_CONFIG = {
    "host": os.getenv("DB_HOST"),
    "port": os.getenv("DB_PORT"),
    "dbname": os.getenv("DB_NAME"),
    "user": os.getenv("DB_USER"),
    "password": os.getenv("DB_PASSWORD"),
}


def consultar_productos():
    conn = psycopg2.connect(**DB_CONFIG)
    try:
        with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
            cur.execute("SELECT id, nombre, precio FROM productos ORDER BY id")
            return [dict(row) for row in cur.fetchall()]
    finally:
        conn.close()


def imprimir_tabla(productos):
    if not productos:
        print("(sin filas)")
        return
    columnas = list(productos[0].keys())
    anchos = {c: max(len(c), max(len(str(p[c])) for p in productos)) for c in columnas}
    print(" | ".join(c.ljust(anchos[c]) for c in columnas))
    print("-+-".join("-" * anchos[c] for c in columnas))
    for p in productos:
        print(" | ".join(str(p[c]).ljust(anchos[c]) for c in columnas))


# Ejecuta el SELECT una vez al arrancar y lo muestra por consola
try:
    productos = consultar_productos()
    print("Conexión a PostgreSQL exitosa. Productos encontrados:")
    imprimir_tabla(productos)
except Exception as error:
    print(f"Error al conectar con PostgreSQL: {error}", file=sys.stderr)


# Además, expone el mismo resultado como endpoint HTTP
@app.get("/productos")
def productos_endpoint():
    try:
        return jsonify(consultar_productos())
    except Exception as error:
        return jsonify({"error": str(error)}), 500


@app.get("/")
def index():
    return "App conectada a PostgreSQL. Probá GET /productos"


if __name__ == "__main__":
    port = int(os.getenv("PORT", "3000"))
    app.run(host="0.0.0.0", port=port)
```

La aplicación hace las dos cosas que pide el enunciado del laboratorio: ejecuta un `SELECT` y muestra el resultado por consola al arrancar, y además lo expone en un endpoint HTTP simple (`GET /productos`) para poder verificarlo también desde un navegador o `curl`.

Ejecuto la aplicación:

```bash
python app.py
```

> [!warning] Al ejecutar la app aparece un error de conexión con la base de datos: `Connection refused` en el puerto 5432
> ![](capturas/Pasted%20image%2020260922201349.png)
> El error se debe a que mi arquitectura tiene **dos VM**: la app corre en `vm-docker-app` y PostgreSQL en `vm-postgres`. Con `DB_HOST=localhost` la app busca PostgreSQL en su propia máquina (`127.0.0.1`), donde no hay nada escuchando en el puerto 5432. Además, PostgreSQL en `vm-postgres` solo escucha en `127.0.0.1` (lo vi antes con `ss -tlnp`), así que tampoco aceptaría conexiones que vengan de otra máquina. Hay que resolver las dos cosas.

### Configurar PostgreSQL para aceptar conexiones remotas (en `vm-postgres`)

Primero edito `postgresql.conf` para que PostgreSQL escuche en todas las interfaces de red y no solo en `localhost`:

```bash
sudo nano /etc/postgresql/18/main/postgresql.conf
```

Busco la línea `listen_addresses` (viene comentada por defecto) y la cambio a:

```
listen_addresses = '*'
```

![](capturas/Pasted%20image%2020260922224519.png)

Escuchar en todas las interfaces no significa que cualquiera pueda entrar. El filtro real de quién se puede autenticar es `pg_hba.conf`:

```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

Al final del archivo agrego una línea que permite **solo** al usuario `labapp` conectarse **solo** a `labdb` y **solo** desde la IP de `vm-docker-app`:

```
# Permitir conexiones desde vm-docker-app (app y contenedores Docker)
host    labdb    labapp    192.168.1.60/32    scram-sha-256
```

> Nota: uso `scram-sha-256` en vez de `md5` porque es el método de autenticación por defecto a partir de PostgreSQL 14, y en la versión 18 `md5` está marcado como obsoleto.

![](capturas/Pasted%20image%2020260922224735.png)

`listen_addresses` necesita un reinicio completo del servicio (no alcanza con `reload`):

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```

Compruebo que ahora el puerto 5432 escucha en todas las interfaces (`0.0.0.0:5432`) y no solo en `127.0.0.1` como antes:

```bash
sudo ss -tlnp | grep 5432
```

![](capturas/Pasted%20image%2020260922224849.png)

Reviso el **firewall**. Si `ufw` aparece como `inactive` no hace falta hacer nada; si está activo, abro el puerto solo para `vm-docker-app`:

```bash
sudo ufw status
sudo ufw allow from 192.168.1.60 to any port 5432
```

![](capturas/Pasted%20image%2020260922224928.png)

### Probar la conexión desde `vm-docker-app`

Antes de volver a lanzar la app, compruebo que desde `vm-docker-app` llego al puerto de PostgreSQL de la otra VM (`192.168.1.61`, la IP fija de `vm-postgres`):

```bash
nc -zv 192.168.1.61 5432
```

![](capturas/Pasted%20image%2020260922225049.png)

Cambio en el `.env` la variable `DB_HOST` para que apunte a la IP de `vm-postgres` en vez de a `localhost`:

```
DB_HOST=192.168.1.61
```

![](capturas/Pasted%20image%2020260922225308.png)

Vuelvo a ejecutar la app con `python app.py` y ahora sí conecta y muestra la tabla de productos por consola:

![](capturas/Pasted%20image%2020260922225353.png)

Desde otra terminal pruebo también el endpoint:

```bash
curl http://localhost:3000/productos
```

Detengo la app con `Ctrl+C` y desactivo el entorno virtual con `deactivate`, ya que a partir de ahora la app va a correr dentro de Docker.

## PASO 6: Contenerizar la aplicación (en `vm-docker-app`) y conectarla a PostgreSQL (en `vm-postrges)

### Crear el `Dockerfile`

Dentro de `~/lab-postgres-docker` creo el archivo `nano Dockerfile` (sin extensión):

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py ./

EXPOSE 3000

CMD ["python", "app.py"]
```

- `FROM python:3.12-slim`: imagen base con Python 3.12 sobre Debian reducido, así la imagen final pesa poco. Esto lo obtengo desde Docker Hub
- `WORKDIR /app`: carpeta de trabajo dentro del contenedor.
- Copio primero `requirements.txt` e instalo las dependencias y **después** copio `app.py`. De esta forma Docker reutiliza la capa de dependencias en caché si solo cambio el código.
- `EXPOSE 3000`: documenta el puerto de la app (publicarlo lo hace `docker run -p`).
- `CMD`: el comando que se ejecuta al arrancar el contenedor.

### Crear el `.dockerignore`

Para que no se copien dentro de la imagen archivos innecesarios o sensibles, creo `nano .dockerignore`:

```
venv
__pycache__
*.pyc
.env
.git
```

> Nota: excluyo `venv` porque las dependencias se instalan dentro del contenedor con `pip install`, y `.env` porque las credenciales de la DDBB no deben quedar guardadas dentro de la imagen: se pasan como variables de entorno al ejecutar el contenedor.

![](capturas/Pasted%20image%2020260923121222.png)

### Construir la imagen

```bash
docker build -t lab-postgres-app .
docker images | grep lab-postgres-app
```

![](capturas/Pasted%20image%2020260923122624.png)

### Primer intento: el contenedor con `DB_HOST=localhost`

Para entender cómo se comporta la red de Docker, primero lanzo el contenedor con `localhost` como host de la base de datos:

```bash
docker run -d --name lab-postgres-app -p 3000:3000 \
  -e DB_HOST=localhost \
  -e DB_PORT=5432 \
  -e DB_NAME=labdb \
  -e DB_USER=labapp \
  -e DB_PASSWORD=labapp_26 \
  lab-postgres-app

docker logs lab-postgres-app
```

- `-d`: ejecuta el contenedor en segundo plano para poder seguir usando la terminal.
- `--name`: nombre del contenedor para usarlo con `docker logs`, `docker stop`, etc.
- `-p 3000:3000`: publica el puerto 3000 del contenedor en el 3000 de la VM.
- `-e`: variables de entorno que la app lee con `os.getenv`.

> [!warning] El contenedor arranca pero no conecta: `Connection refused` en `127.0.0.1`
> ![](capturas/Pasted%20image%2020260923122933.png)
> Dentro de un contenedor, `localhost` es el propio contenedor, no la VM donde corre Docker. Cada contenedor tiene su propia red aislada, así que el `localhost` del contenedor y el de la VM son dos cosas distintas aunque estén en la misma máquina física.

Elimino este intento antes de continuar y vuelvo a intentar corrigiendo la variable de entorno `DB_HOST=192.168.1.61` que apunte directamente a la VM  de PostgreSQL:

```bash
docker stop lab-postgres-app
docker rm lab-postgres-app
```

Verifico la conexión:

```bash
docker run -d --name lab-postgres-app -p 3000:3000 \
  -e DB_HOST=192.168.1.61 \
  -e DB_PORT=5432 \
  -e DB_NAME=labdb \
  -e DB_USER=labapp \
  -e DB_PASSWORD=labapp_26 \
  lab-postgres-app
  
docker ps
docker logs lab-postgres-app
```

![](capturas/Pasted%20image%2020260923123257.png)

Compruebo también el endpoint desde mi Windows:

http://192.168.1.60:3000/productos

![](Laboratorios/ALTEN-Group-Practicas-2026/lab-postgres-docker/capturas/Pasted%20image%2020260923125019.png)

Con esto queda cumplido el objetivo del laboratorio: una aplicación corriendo dentro de un contenedor Docker que consulta datos de un PostgreSQL instalado como servicio nativo (no en contenedor).

## PASO 7: Conectar el contenedor a PostgreSQL 

### Investigación: cómo accede un contenedor a otros servicios

Investigando encontré tres formas habituales de que un contenedor llegue a un servicio que está fuera de él:

- **`host.docker.internal`**: nombre especial que apunta a la máquina donde corre Docker. En Docker Desktop (Windows/Mac) funciona solo; en Linux hay que habilitarlo con `--add-host=host.docker.internal:host-gateway`.
- **IP del bridge `docker0`** (normalmente `172.17.0.1`): la puerta de enlace de la red de contenedores, que también es la propia máquina donde corre Docker.
- **`--network host`**: el contenedor comparte la red de la VM y pierde su aislamiento.

Las tres sirven para llegar a un servicio **de la misma máquina donde corre Docker**. En mi caso PostgreSQL está en **otra VM** (`vm-postgres`), así que lo que necesito es usar directamente su IP de la red local. Los contenedores en la red `bridge` pueden salir a la red local sin configuración extra, porque Docker hace NAT: el tráfico sale del contenedor (`172.17.0.x`) y llega a `vm-postgres` con la IP de `vm-docker-app` (`192.168.1.60`). Por eso la línea que agregué antes en `pg_hba.conf` también sirve para el contenedor.

Puedo comprobar la subred que usa Docker con:

```bash
docker network inspect bridge | grep Subnet
```

![](capturas/Pasted%20image%2020260923124258.png)

> Nota: si PostgreSQL estuviera instalado en la misma VM que Docker , habría que lanzar el contenedor con `--add-host=host.docker.internal:host-gateway` y `-e DB_HOST=host.docker.internal`, y en `pg_hba.conf` permitir la subred de Docker (`172.17.0.0/16`) en vez de la IP de la otra VM.

## Conclusión

Lo que más me costó entender fue que `localhost` no es siempre el mismo sitio: depende de dónde se ejecuta el proceso (mi VM, otra VM o un contenedor). También aprendí que en PostgreSQL abrir la red tiene dos capas: `listen_addresses` decide en qué interfaces escucha y `pg_hba.conf` decide quién puede autenticarse y desde dónde.


