# lab-postgres-docker

**Objetivo**
Familiarizarse con:
- Linux (uso básico)
- Docker (instalación y uso de contenedor)
- PostgreSQL
- Conexión entre servicios
---
Tareas
1. Instalación de entorno
- Instalar Ubuntu 24
- Crear un usuario de trabajo (no root)
- Actualizar el sistema
---
2. Docker
- Instalar Docker
- Agregar su usuario al grupo "docker" (para no usar sudo)
**sudo usermod -aG docker $USER**
Otro tip...no se olviden de crear el grupo docker.
- Validar ejecución:
docker run hello-world
---
3. PostgreSQL
- Instalar PostgreSQL (Rita solicito la 18) se debe instalar como servicio
- Verificar que esté corriendo
- Crear
- una base de datos
- un usuario
- una tabla simple
- Insertar algunos datos de prueba
---
4. Aplicación en Docker.
- Crear una aplicación simple (puede ser Node.js o la tecnologia que conozcan)
- Ejecutarla dentro de un contenedor Docker
La aplicación debe:
- conectarse a PostgreSQL
- ejecutar un SELECT
- mostrar el resultado (consola o endpoint simple)
---
5. Conexión
- Lograr que el contenedor se conecte a PostgreSQL instalado en el host
A tener en cuenta:
Investigar cómo un contenedor accede a servicios del host (ej: "host.docker.internal" o IP)

---
6. Documentación
Documentar en un archivo:
- pasos realizados
- problemas encontrados
- cómo los resolvieron
---
Requisito deseable:
- Crear repo git del proyecto que sera parte de la documentación con los respectivos commits.
- No copiar/pegar sin entender — la idea es que investiguen y prueben
---
El objetivo final es lograr que una app corriendo en Docker consulte datos de PostgreSQL instalado en el mismo servidor.
Cualquier duda, me encuentro a su disposición. La idea es que aprendan en el proceso sobre un entorno basico.