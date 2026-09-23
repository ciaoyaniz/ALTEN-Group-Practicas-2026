## PASO 7: Conectar el contenedor a PostgreSQL (en `vm-docker-app`)

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

### Ejecutar el contenedor apuntando a `vm-postgres`

```bash
docker run -d --name lab-postgres-app -p 3000:3000 \
  -e DB_HOST=192.168.1.61 \
  -e DB_PORT=5432 \
  -e DB_NAME=labdb \
  -e DB_USER=labapp \
  -e DB_PASSWORD=labapp_26 \
  lab-postgres-app
```

La única diferencia con el intento anterior es `DB_HOST`, que ahora apunta a la IP de `vm-postgres` en vez de a `localhost`.

### Verificar la conexión

```bash
docker ps
docker logs lab-postgres-app
```

> **[CAPTURA 9]** `docker ps` con el contenedor `Up` y `docker logs` mostrando la tabla de productos.

Compruebo también el endpoint:

```bash
curl http://localhost:3000/productos
```

> **[CAPTURA 10]** `curl` devolviendo el JSON desde el contenedor.

Como la VM está en modo puente, también puedo abrirlo desde el navegador de mi host Windows en `http://192.168.1.60:3000/productos`.

> **[CAPTURA 11]** Navegador de Windows mostrando el JSON de `/productos`.

Con esto queda cumplido el objetivo del laboratorio: una aplicación corriendo dentro de un contenedor Docker que consulta datos de un PostgreSQL instalado como servicio nativo (no en contenedor).

> Nota: si PostgreSQL estuviera instalado en la misma VM que Docker (como plantea el tutorial), habría que lanzar el contenedor con `--add-host=host.docker.internal:host-gateway` y `-e DB_HOST=host.docker.internal`, y en `pg_hba.conf` permitir la subred de Docker (`172.17.0.0/16`) en vez de la IP de la otra VM.

## PASO 8: Repositorio Git

Convierto la carpeta del laboratorio en un repositorio Git. El `.gitignore` está configurado para subir **solo la documentación** (archivos `.md` y la carpeta `capturas`), de modo que el código, el `Dockerfile` y sobre todo el `.env` con la contraseña de la DDBB quedan fuera del repositorio.

```bash
git init
git add .gitignore practica.md solución.md capturas/
git commit -m "Documenta instalacion de Ubuntu, Docker y PostgreSQL 18"
```

A medida que avanzo con el laboratorio voy haciendo commits pequeños que reflejan cada paso, en vez de uno solo al final:

```bash
git add solución.md capturas/
git commit -m "Documenta la conexion del contenedor con PostgreSQL de vm-postgres"
```

Reviso el historial y compruebo que el `.env` nunca se haya subido:

```bash
git log --oneline
git log --all --full-history -- .env
```

> **[CAPTURA 12]** `git log --oneline` con la lista de commits.

## Problemas encontrados y cómo los resolví

### Problema 1: incompatibilidad de las boxes de Vagrant con PostgreSQL 18
- **Síntoma**: problemas al instalar PostgreSQL 18 sobre las VM creadas con Vagrant y la box `bento/ubuntu-24.04`.
- **Causa**: compatibilidad de la box con los paquetes del repositorio de PostgreSQL.
- **Solución**: rehice las VM en VirtualBox a partir de la ISO oficial de Ubuntu 24.x.

### Problema 2: la app no conectaba a PostgreSQL (`Connection refused` en `127.0.0.1`)
- **Síntoma**: `connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused` al ejecutar `python app.py`.
- **Causa**: la app corre en `vm-docker-app` pero PostgreSQL está en `vm-postgres`; `localhost` apuntaba a la VM equivocada.
- **Solución**: cambiar `DB_HOST` a la IP de `vm-postgres`.

### Problema 3: PostgreSQL solo aceptaba conexiones locales
- **Síntoma**: `ss -tlnp` mostraba el puerto 5432 escuchando solo en `127.0.0.1`.
- **Causa**: por seguridad, PostgreSQL recién instalado solo escucha en `localhost` y `pg_hba.conf` no permite conexiones de otras máquinas.
- **Solución**: `listen_addresses = '*'` en `postgresql.conf`, una línea en `pg_hba.conf` limitada a `labapp`, `labdb` y la IP de `vm-docker-app`, y reinicio del servicio.

### Problema 4: el contenedor no conectaba con `DB_HOST=localhost`
- **Síntoma**: `docker logs` mostraba `Connection refused` en `127.0.0.1`.
- **Causa**: dentro del contenedor `localhost` es el propio contenedor, que tiene su red aislada.
- **Solución**: pasar al contenedor la IP de `vm-postgres` con `-e DB_HOST=...`.

## Conclusión

Lo que más me costó entender fue que `localhost` no es siempre el mismo sitio: depende de dónde se ejecuta el proceso (mi VM, otra VM o un contenedor). También aprendí que en PostgreSQL abrir la red tiene dos capas: `listen_addresses` decide en qué interfaces escucha y `pg_hba.conf` decide quién puede autenticarse y desde dónde.


