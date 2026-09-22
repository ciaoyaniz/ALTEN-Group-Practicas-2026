# Solución

## PASO 1: Configuraciones generales

### Instalar Ubuntu 24 y preparar el entorno

> [!WARNING] Cambio de entorno
> Al comienzo de este laboratorio cree VMs con Vagrant y boxes `bento/ubuntu-24.04`, pero, debido a problemas de compatibilidad al instalar PostgreSQL v-18, decidí rehacer las VMs con sus ISO oficiales de Ubuntu 24.x.

![](capturas/Pasted%20image%2020260921234441.png)

Creación de dos VM con Ubuntu 24.04 LTS (el usuario de trabajo no root que voy a usar lo he creado durante la instalación, llamado `ciaoyaniz`). 

Además he habilitado los servicios SSH para poder conectarme de forma remota a ellas.

> **Red**: en modo Bridged (Adaptador Puente), la VM se comporta como una máquina más de la red local (192.168.1.0), para poder ver a la otra VM por IP.

### Actualización del sistema

Inicié sesión desde mi host anfitrión mediante el servicio SSH con el comando `ssh ciaoyaniz@192.168.1.x`. Previamente obtuve la IP asignada a cada VM con el comando `ip a`. 

Una vez dentro de cada sistema, los actualicé:
```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

![](capturas/Pasted%20image%2020260921223035.png)

![](capturas/Pasted%20image%2020260921223051.png)

### Asignar permisos `sudo` a mi usuario

`usermod -aG sudo` agrega mi usuario al grupo `sudo`, habilitándome a ejecutar comandos administrativos con `sudo` sin ser `root`.

![](capturas/Pasted%20image%2020260921224242.png)

El proceso lo he repetido en las dos VM.

## PASO 2: Configurar VM con Docker
### Instalar Docker

He creado un script con ayuda de IA para poder instalar de forma rápida el servicio de Docker con sus dependencias y claves GPG ya que este es un paso crucial y no quería detenerme en ello.

Para crearlo y ejecutarlo utilice los comandos de edición, permisos de ejecución y al final ejecución del script:

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

![](capturas/Pasted%20image%2020260921234941.png)

El proceso de instalación ha sido un poco lento pero el script ha funcionado correctamente:

![](capturas/Pasted%20image%2020260922124241.png)

Levanté un contenedor de prueba para verificar:

![](capturas/Pasted%20image%2020260922001758.png)

## PASO 3: Configurar VM con PostgreSQL

### Instalar PostgreSQL sobre Ubuntu

Primero agregué el repositorio oficial de PostgreSQL y actualicé la lisat de paquetes disponibles para luego instalarlos:

```bash
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
  --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
> /etc/apt/sources.list.d/pgdg.list'

sudo apt update
```

Instalar PostgreSQL 18:

```bash
sudo apt install -y postgresql-18 postgresql-client-18
```

Luego de reiniciar el servicio compruebo que está habilitado y funcionando:

![](capturas/Pasted%20image%2020260922125646.png)

PostgreSQL utiliza clusters para poder instalar varias instancias SQL:

![](capturas/Pasted%20image%2020260922130114.png)

El resultado muestra un clúster con PostgreSQL v18

### Ingresar al prompt por terminal

Para verificar la version del servicio se puede ingresar al prompt directamente y verlo desde ahí pero para hacerlo necesito ingresar con el usuario que ha creado previamente PostgreSQL llamado `postgres`.

```bash
sudo -u postgres psql
SELECT version();
```

![](capturas/Pasted%20image%2020260922130940.png)

## PASO 4: Crear base de datos

### Crear base de datos y usuario de servicio

Una vez dentro de la terminal creo la base de datos que va a servir como almacenamiento de mi app en Python

Crear base de datos:

```sql
CREATE DATABASE labdb;
```

![](capturas/Pasted%20image%2020260922141646.png)

La base de datos va a tener un usuario de servicio que va a mantener el servicio activo, conectarse y operar sobre la DDBB

```sql
CREATE USER labapp WITH PASSWORD 'labapp_26';
GRANT ALL PRIVILEGES ON DATABASE labdb TO labapp;
```

Los permisos de PostgreSQL se gestionan a nivel de base de datos y esquemas, por lo tanto hay que asignarle los permisos de esquema `public` que faltan al usuario. Dentro de la configuración de la DDBB:

```SQL
\c labdb
GRANT ALL ON SCHEMA public TO labapp;
```
Una vez activado, el usuario puede conectarse y operar sobre la base de datos. 

> Nota: `public` es una carpeta lógica dentro de una base de datos donde se guardan tablas, vistas, secuencias, etc. Por motivos de seguridad estos permisos han sido restringidos a partir de la version 15. 

Dar permisos sobre tabla:
```sql
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO labapp;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO labapp;
```

Estos permisos sirven para que mi user pueda leer datos que hayan creado otros usuario en la base de datos, no solo los que yo he ingresado.

### Crear una tabla de ejemplo

```sql
CREATE TABLE productos (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    precio NUMERIC(10, 2) NOT NULL,
    creado_en TIMESTAMP DEFAULT NOW()
);
```

```sql
INSERT INTO productos (nombre, precio) VALUES
    ('Teclado mecánico', 45.99),
    ('Mouse inalámbrico', 19.50),
    ('Monitor 24 pulgadas', 149.00),
    ('Webcam HD', 32.75);
```

![](capturas/Pasted%20image%2020260922150315.png)


