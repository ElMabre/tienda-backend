Backend — Tienda Perritos 
Microservicios de Ventas y Despachos desarrollados con Java 17 + Spring Boot 3.4.x, contenedorizados con Docker y desplegados automáticamente en AWS EC2 mediante GitHub Actions.

Tecnologías
ComponenteVersiónJava17Spring Boot3.4.xMySQL8.0Docker Enginev24+Docker Composev2+

Estructura del proyecto
Cada microservicio incluye su propio Dockerfile con construcción multi-stage:

Etapa 1 (build): imagen Maven/Java completa para compilar y generar el .jar.
Etapa 2 (producción): imagen ligera eclipse-temurin:17-jre-alpine con solo el artefacto ejecutable.

El proceso corre bajo un usuario no-root (springuser) para reducir la superficie de ataque en EC2.

Variables de entorno
El proyecto no hardcodea credenciales. Todas las configuraciones se inyectan por variables de entorno:
VariableDescripciónValor por defecto (local)DB_ENDPOINTHost de la base de datosdbDB_PORTPuerto MySQL3306DB_NAMENombre de la base de datostienda_perritosDB_USERNAMEUsuario MySQLrootDB_PASSWORDContraseña MySQLadmin123
En AWS, DB_ENDPOINT se reemplaza por la IP privada de la instancia EC2-DB.

Levantar en local
bash# Construir imágenes y levantar el stack
docker compose up -d --build

# Verificar estado de los servicios
docker compose ps

# Ver logs en tiempo real
docker compose logs -f
Healthcheck y dependencias
El docker-compose.yml define un healthcheck sobre MySQL (mysqladmin ping). Los microservicios de Spring Boot no arrancan hasta que la base de datos reporta healthy, evitando errores de conexión en el inicio.

Persistencia de datos
Se usa un named volume (dbdata) mapeado a /var/lib/mysql en el contenedor de base de datos.
Se eligió named volume por sobre bind mount porque:

Es gestionado directamente por el Docker daemon, sin depender de rutas del sistema operativo anfitrión.
Funciona de forma idéntica en Fedora (local) y Amazon Linux 2023 (EC2).
Ofrece mejor rendimiento de I/O sobre volúmenes EBS de AWS.

Los datos persisten ante reinicios del contenedor o de la instancia EC2.

Pipeline CI/CD
El archivo .github/workflows/deploy.yml se activa con cada push a la rama deploy.
Flujo:

Autenticación en AWS usando los secrets AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY y AWS_SESSION_TOKEN.
Construcción de las imágenes Docker de cada microservicio.
Publicación en Amazon ECR bajo los tags ventas-latest y despachos-latest.
Despliegue remoto vía AWS SSM: la instancia EC2-Backend descarga las nuevas imágenes, detiene los contenedores anteriores de forma ordenada y levanta los actualizados con las variables de entorno correspondientes.

Solución de errores frecuentes
Public Key Retrieval is not allowed — resuelto agregando allowPublicKeyRetrieval=true en la cadena de conexión JDBC del application.properties.
Error de zona horaria — Se resolvio forzando serverTimezone=UTC en la misma cadena de conexión.
