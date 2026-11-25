# Trabajo final · Corte 3 · Sistemas Operativos 103027

**Universidad Autónoma de Manizales (UAM)**  
**Asignatura:** Sistemas Operativos  
**Proyecto:** Despliegue de frontend en AWS y backend en Azure  
**Docente:** Cristian David Gutiérrez Bedoya

---

## 1. Descripción del proyecto

El objetivo de este proyecto es desplegar una aplicación web distribuida en dos servidores diferentes: un frontend alojado en AWS EC2 y un backend con base de datos PostgreSQL alojado en Microsoft Azure.

La aplicación permite consultar una tabla de sistemas operativos mediante una API REST desarrollada en Flask.

El frontend realiza las solicitudes a la API y presenta los datos almacenados en PostgreSQL.

El proyecto utiliza dos subdominios independientes:

| Componente | Plataforma | Dirección |
|---|---|---|
| Frontend | AWS EC2 | Pendiente de confirmar |
| Backend API | Azure VM | https://api-anyu.scaleautomatization.tech |

## 2. Arquitectura del sistema

La arquitectura está distribuida de la siguiente manera:

```text
                 USUARIO
                    |
                    v
          FRONTEND - AWS EC2
             Ubuntu 24.04
                 Nginx
                    |
                    | HTTPS
                    |
                    v
          BACKEND - AZURE VM
             Ubuntu 24.04
                 Nginx
                    |
                    | HTTP interno
                    |
                    v
             DOCKER COMPOSE
                    |
             +------+------+
             |             |
             v             v
       CONTENEDOR     CONTENEDOR
        BACKEND          DB
         Flask        PostgreSQL
         Gunicorn       16
             |             |
             +------+------+
                    |
              VOLUMEN
           datos_postgres
```

### Distribución de responsabilidades

**AWS EC2 - Frontend**

- Servidor Ubuntu 24.04.
- Nginx instalado directamente en el sistema operativo.
- Publicación de archivos estáticos HTML y JavaScript.
- Configuración de la URL pública de la API.
- Certificado HTTPS mediante Certbot.

**Azure VM - Backend**

- Servidor Ubuntu 24.04.
- Docker y Docker Compose.
- Backend desarrollado en Flask.
- Gunicorn como servidor de aplicaciones.
- PostgreSQL 16 como base de datos.
- Nginx como proxy inverso.
- Certificado HTTPS mediante Let's Encrypt y Certbot.

## 3. Estructura del repositorio

```text
proyecto-SO/
│
├── backend/
│   ├── app.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .env.example
│
├── db/
│   └── init.sql
│
├── frontend/
│   ├── index.html
│   └── config.js
│
├── deploy/
│   ├── nginx-backend.conf
│   └── nginx-frontend.conf
│
├── compose.yaml
├── .env
├── .gitignore
└── README.md
```

**Importante:** el archivo `.env` contiene información sensible y no debe subirse a GitHub. Para compartir la configuración se utiliza `.env.example`.

## 4. Tecnologías utilizadas

| Tecnología | Función |
|---|---|
| Ubuntu Server 24.04 | Sistema operativo de los servidores |
| AWS EC2 | Alojamiento del frontend |
| Microsoft Azure | Alojamiento del backend |
| Python | Lenguaje del backend |
| Flask | Framework para desarrollar la API |
| Gunicorn | Servidor de aplicaciones WSGI |
| PostgreSQL 16 | Sistema gestor de base de datos |
| Docker | Contenedores |
| Docker Compose | Administración de servicios |
| Nginx | Servidor web y proxy inverso |
| Certbot | Gestión de certificados HTTPS |
| Let's Encrypt | Autoridad de certificados SSL/TLS |
| Git y GitHub | Control de versiones |
| Docker Hub | Publicación de la imagen del backend |

## 5. Endpoints de la API

El backend dispone de dos endpoints principales.

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/salud` | Comprueba que el backend y la base de datos respondan |
| GET | `/api/sistemas` | Devuelve los sistemas operativos almacenados en PostgreSQL |

### Prueba de salud

```bash
curl https://api-anyu.scaleautomatization.tech/api/salud
```

Respuesta esperada:

```json
{
  "estado": "ok"
}
```

### Consulta de sistemas operativos

```bash
curl https://api-anyu.scaleautomatization.tech/api/sistemas
```

La respuesta contiene los ocho sistemas operativos registrados mediante `db/init.sql`.

## 6. Ejecución del proyecto en local

Para realizar pruebas locales se necesita:

- Docker.
- Python 3.10 o superior.
- Git.

Se utilizan tres terminales desde la raíz del proyecto.

### 6.1. Base de datos PostgreSQL

En la primera terminal:

```bash
docker run --rm --name so-db -p 5432:5432 \
  -e POSTGRES_USER=so \
  -e POSTGRES_PASSWORD=so \
  -e POSTGRES_DB=so \
  -v "$PWD/db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro" \
  postgres:16
```

El archivo `init.sql` crea las tablas y carga los datos iniciales.

Esta ejecución local no utiliza un volumen persistente, por lo que al eliminar el contenedor se pierden sus datos.

### 6.2. Backend Flask

En la segunda terminal:

```bash
cd backend
```

Crear el entorno virtual:

```bash
python -m venv .venv
```

Activarlo en Windows:

```powershell
.venv\Scripts\activate
```

En Linux:

```bash
source .venv/bin/activate
```

Instalar las dependencias:

```bash
pip install -r requirements.txt
```

Configurar las variables de entorno.

En Linux:

```bash
export DATABASE_URL=postgresql://so:so@localhost:5432/so
export ORIGEN_PERMITIDO=http://localhost:8080
```

En PowerShell:

```powershell
$env:DATABASE_URL="postgresql://so:so@localhost:5432/so"
$env:ORIGEN_PERMITIDO="http://localhost:8080"
```

Iniciar Flask:

```bash
flask --app app run --port 8000
```

Comprobar:

```bash
curl http://localhost:8000/api/salud
curl http://localhost:8000/api/sistemas
```

### 6.3. Frontend

En la tercera terminal:

Modificar temporalmente `frontend/config.js`:

```javascript
const API_URL = "http://localhost:8000";
```

Iniciar el servidor local:

```bash
cd frontend
python -m http.server 8080
```

Abrir en el navegador:

```text
http://localhost:8080
```

La página debe mostrar la tabla de sistemas operativos.

Si aparece un error de CORS, verificar que `ORIGEN_PERMITIDO` coincida exactamente con el origen del frontend.

Antes del despliegue, configurar nuevamente la URL pública del backend en `config.js`.

## 7. Construcción de la imagen Docker del backend

El backend utiliza un Dockerfile basado en una imagen oficial de Python con versión fija.

Ubicación:

```text
backend/Dockerfile
```

Construir la imagen desde la raíz del repositorio:

```bash
docker build -t andrewbar1244/so-backend:1.0 -f backend/Dockerfile backend
```

Verificar la imagen:

```bash
docker images
```

La imagen utilizada en el proyecto es:

```text
andrewbar1244/so-backend:1.0
```

## 8. Publicación en Docker Hub

La imagen se publicó en Docker Hub para que Azure pueda descargarla sin necesidad de construirla nuevamente en la máquina virtual.

Iniciar sesión:

```bash
docker login
```

Publicar:

```bash
docker push andrewbar1244/so-backend:1.0
```

Repositorio de la imagen:

https://hub.docker.com/r/andrewbar1244/so-backend

La máquina virtual de Azure utiliza esta imagen mediante la propiedad `image:` del archivo `compose.yaml`.

## 9. Configuración de variables de entorno

El proyecto utiliza variables de entorno para configurar la conexión a PostgreSQL y el origen permitido para las solicitudes del frontend.

Crear el archivo `.env` en la raíz del proyecto, tomando como referencia `.env.example`.

Ejemplo de configuración:

```env
POSTGRES_USER=so
POSTGRES_PASSWORD=CAMBIAR_POR_UNA_CLAVE_SEGURA
POSTGRES_DB=so

DATABASE_URL=postgresql://so:CAMBIAR_POR_UNA_CLAVE_SEGURA@db:5432/so

ORIGEN_PERMITIDO=https://DOMINIO-FRONTEND-AWS
```

**Consideraciones:**

- `POSTGRES_USER`: usuario de PostgreSQL.
- `POSTGRES_PASSWORD`: contraseña de PostgreSQL.
- `POSTGRES_DB`: nombre de la base de datos.
- `DATABASE_URL`: cadena de conexión utilizada por Flask.
- `ORIGEN_PERMITIDO`: dirección HTTPS autorizada para realizar solicitudes a la API.

El nombre de host `db` corresponde al servicio PostgreSQL dentro de la red de Docker Compose.

El valor de `ORIGEN_PERMITIDO` debe actualizarse con el dominio definitivo del frontend cuando esté disponible.

Proteger el archivo:

```bash
chmod 600 .env
```

No publicar el archivo `.env` en GitHub.

## 10. Despliegue del backend en Azure

### 10.1. Máquina virtual

El backend está desplegado en una máquina virtual con:

| Característica | Configuración |
|---|---|
| Plataforma | Microsoft Azure |
| Sistema operativo | Ubuntu Server 24.04 LTS |
| Usuario SSH | azureuser |
| IP privada | 172.16.0.4 |
| IP pública | 172.149.24.44 |
| Contenedores | Docker |
| Orquestación | Docker Compose |

### 10.2. Instalación de Docker

Instalar Docker:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Permitir que el usuario administre Docker:

```bash
sudo usermod -aG docker azureuser
```

Aplicar los permisos a la sesión:

```bash
newgrp docker
```

Verificar:

```bash
docker --version
docker compose version
```

### 10.3. Obtener el proyecto

Clonar el repositorio:

```bash
git clone -b develop https://github.com/oteroyulian461-alt/proyecto-SO.git
```

Entrar al directorio:

```bash
cd proyecto-SO
```

Crear el archivo `.env` con los valores correspondientes al servidor.

### 10.4. Levantar los servicios

Ejecutar:

```bash
docker compose pull
docker compose up -d
```

Verificar:

```bash
docker compose ps
```

Los servicios deben aparecer en ejecución:

- `so-backend`
- `so-db`

PostgreSQL debe alcanzar el estado `healthy`.

## 11. Configuración de PostgreSQL y persistencia

La base de datos utiliza PostgreSQL 16 sobre Docker.

El archivo `db/init.sql` se monta en:

```text
/docker-entrypoint-initdb.d/
```

De esta forma, PostgreSQL ejecuta el script de inicialización cuando se crea una base de datos nueva con un directorio de datos vacío.

El servicio utiliza un volumen persistente:

```text
proyecto-so_datos_postgres
```

Este volumen permite conservar la información cuando los contenedores se reinician.

Comprobar los volúmenes:

```bash
docker volume ls
```

Verificar el estado de los contenedores:

```bash
docker compose ps
```

Reiniciar los servicios:

```bash
docker compose restart
```

Comprobar nuevamente:

```bash
docker compose ps
```

**Importante:** no utilizar `docker compose down -v` durante las pruebas de persistencia, porque eliminaría el volumen de PostgreSQL.

La base de datos no publica su puerto hacia Internet. Su comunicación se realiza internamente mediante la red de Docker Compose.

## 12. Configuración de Nginx en Azure

Nginx está instalado directamente en Ubuntu y funciona como proxy inverso.

Su responsabilidad es recibir las solicitudes HTTP/HTTPS y enviarlas al backend Flask que escucha en el puerto 8000.

Archivo de configuración:

```text
/etc/nginx/sites-available/so-backend
```

El repositorio conserva una copia en:

```text
deploy/nginx-backend.conf
```

La configuración utiliza:

```nginx
proxy_pass http://127.0.0.1:8000;
```

Esto permite que Nginx se comunique con Gunicorn sin exponer el puerto 8000 públicamente.

Comprobar la configuración:

```bash
sudo nginx -t
```

Recargar Nginx:

```bash
sudo systemctl reload nginx
```

Consultar el estado:

```bash
sudo systemctl status nginx --no-pager
```

## 13. Configuración HTTPS con Certbot

El backend utiliza un certificado SSL/TLS gratuito emitido por Let's Encrypt.

Dominio del backend:

```text
https://api-anyu.scaleautomatization.tech
```

Instalar Certbot y el complemento de Nginx:

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

Solicitar el certificado:

```bash
sudo certbot --nginx -d api-anyu.scaleautomatization.tech
```

Certbot configura automáticamente el certificado en Nginx y permite redirigir las conexiones HTTP hacia HTTPS.

Rutas del certificado:

```text
/etc/letsencrypt/live/api-anyu.scaleautomatization.tech/fullchain.pem
/etc/letsencrypt/live/api-anyu.scaleautomatization.tech/privkey.pem
```

Verificar los certificados:

```bash
sudo certbot certificates
```

Comprobar la renovación automática:

```bash
sudo certbot renew --dry-run
```

La prueba de renovación se realizó correctamente.

El certificado vigente tiene fecha de expiración del 22 de febrero de 2027 y Certbot configuró su renovación automática.

## 14. Configuración de seguridad

El servidor Azure utiliza dos niveles de protección:

- Firewall UFW de Ubuntu.
- Network Security Group (NSG) de Azure.

### 14.1. Reglas de UFW

Consultar:

```bash
sudo ufw status verbose
```

Reglas configuradas:

| Puerto | Servicio | Acción |
|---|---|---|
| 22 | SSH | Allow |
| 80 | HTTP | Allow |
| 443 | HTTPS | Allow |
| 5432 | PostgreSQL | Sin acceso público |
| 8000 | Backend | Sin acceso público |

La política predeterminada de UFW deniega las conexiones entrantes y permite las salientes.

### 14.2. Reglas del NSG de Azure

Se habilitaron los siguientes puertos de entrada:

| Puerto | Protocolo | Servicio |
|---|---|---|
| 22 | TCP | SSH |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |

No se habilitaron los puertos 5432 ni 8000 en las reglas de entrada del NSG.

### 14.3. Protección de información sensible

El archivo `.env` contiene las credenciales de la base de datos y debe permanecer fuera del repositorio.

Verificar el estado de Git:

```bash
git status
```

Comprobar que `.env` esté incluido en `.gitignore`.

## 15. Verificación del backend desplegado

El backend se puede probar desde la máquina virtual Azure o desde cualquier equipo con acceso a Internet.

### 15.1. Verificar Nginx

```bash
sudo nginx -t
```

### 15.2. Verificar contenedores

```bash
docker compose ps
```

### 15.3. Verificar salud de la API

```bash
curl https://api-anyu.scaleautomatization.tech/api/salud
```

Respuesta esperada:

```json
{
  "estado": "ok"
}
```

### 15.4. Consultar sistemas operativos

```bash
curl https://api-anyu.scaleautomatization.tech/api/sistemas
```

Debe devolver los ocho sistemas operativos registrados en PostgreSQL.

### 15.5. Verificar redirección HTTPS

```bash
curl -I http://api-anyu.scaleautomatization.tech
```

La respuesta esperada es:

```text
HTTP/1.1 301 Moved Permanently
Location: https://api-anyu.scaleautomatization.tech/
```

### 15.6. Consultar registros del backend

```bash
docker compose logs --tail=50 backend
```

Consultar registros de PostgreSQL:

```bash
docker compose logs --tail=50 db
```

## 16. Integración entre AWS y Azure

La integración final requiere que el frontend alojado en AWS pueda consultar la API publicada en Azure.

### 16.1. Configuración del frontend

En AWS se debe modificar:

```text
frontend/config.js
```

La variable `API_URL` debe apuntar a:

```javascript
const API_URL = "https://api-anyu.scaleautomatization.tech";
```

### 16.2. Configuración CORS en Azure

El backend utiliza la variable:

```env
ORIGEN_PERMITIDO=https://DOMINIO-FRONTEND-AWS
```

Esta variable debe contener exactamente el origen HTTPS del frontend.

Una vez definido el dominio de AWS, actualizar `.env` en Azure y recrear únicamente el backend:

```bash
docker compose up -d --force-recreate backend
```

Comprobar los registros:

```bash
docker compose logs --tail=50 backend
```

### 16.3. Prueba de integración

Abrir el dominio del frontend en el navegador.

La página debe consultar la API de Azure y mostrar la tabla con los ocho sistemas operativos.

Para verificar la comunicación:

1. Abrir las herramientas del navegador con F12.
2. Entrar en la pestaña Network.
3. Recargar la página.
4. Buscar la solicitud `/api/sistemas`.
5. Verificar que responda correctamente.

La integración se considera correcta cuando la tabla muestra los datos de PostgreSQL y no aparecen errores de CORS ni Mixed Content.

**Nota:** el dominio definitivo del frontend y el valor final de `ORIGEN_PERMITIDO` deben completarse cuando se confirme la configuración de AWS.

## 17. Configuración del frontend en AWS EC2

Esta parte corresponde al despliegue del frontend en una instancia EC2.

El procedimiento consiste en:

1. Crear una instancia Ubuntu Server 24.04.
2. Configurar las reglas de seguridad para los puertos 22, 80 y 443.
3. Instalar Nginx directamente en Ubuntu.
4. Clonar el repositorio de GitHub.
5. Copiar los archivos de `frontend/` a una carpeta de publicación de Nginx.
6. Configurar `server_name` con el dominio asignado.
7. Activar el sitio en Nginx.
8. Validar la configuración con `sudo nginx -t`.
9. Instalar y configurar Certbot.
10. Emitir el certificado HTTPS.

Archivo de referencia:

```text
deploy/nginx-frontend.conf
```

La configuración final debe utilizar el dominio asignado al frontend y servir los archivos estáticos de la aplicación.

En AWS no se utiliza Docker para el frontend, de acuerdo con las instrucciones del proyecto.

## 18. Solución de problemas

| Problema | Posible causa | Solución |
|---|---|---|
| Error CORS | El origen del frontend no coincide con `ORIGEN_PERMITIDO` | Actualizar la variable y recrear el backend |
| Mixed Content | El frontend HTTPS consulta una API HTTP | Utilizar la URL HTTPS |
| 502 Bad Gateway | Nginx no encuentra el backend | Comprobar contenedores y puerto 8000 |
| Error 500 en `/api/salud` | Problemas de conexión con PostgreSQL | Revisar `DATABASE_URL` y logs |
| Tabla inexistente | No se ejecutó `init.sql` | Comprobar la inicialización de PostgreSQL |
| NXDOMAIN | Registro DNS inexistente o incorrecto | Revisar los registros DNS |
| ERR_CONNECTION_TIMED_OUT | Puerto bloqueado | Revisar NSG y UFW |
| Error de autenticación PostgreSQL | Contraseña diferente entre PostgreSQL y `DATABASE_URL` | Sincronizar las credenciales |
| Certbot no emite certificado | DNS incorrecto o puerto 80 inaccesible | Verificar DNS y reglas de entrada |

## 19. Evidencias para la entrega

Se deben recopilar las siguientes capturas:

### Parte 1. Frontend AWS

- Instancia EC2 funcionando.
- Configuración de Nginx.
- Resultado de `sudo nginx -t`.
- Página publicada con HTTPS y candado.

### Parte 2. Backend Azure

- Imagen publicada en Docker Hub.
- Contenedores funcionando mediante `docker compose ps`.
- PostgreSQL en estado `healthy`.
- Prueba de `/api/salud`.
- Prueba de `/api/sistemas`.
- Configuración Nginx.
- Certificado HTTPS activo.

### Integración

- Página mostrando los ocho sistemas operativos.
- Pestaña Network del navegador mostrando la solicitud a la API.
- Respuesta correcta del endpoint `/api/sistemas`.

### Solución de errores

Documentar un error que haya requerido trabajo para solucionarse y explicar cómo se resolvió.

## 20. Preguntas para la sustentación

**1. En EC2 instalaron Nginx directamente y en Azure usaron Docker. ¿Qué ventaja y qué desventaja vieron en cada forma?**

En AWS, instalar Nginx directamente permite servir archivos estáticos sin agregar una capa de contenedores, aunque requiere administrar las dependencias y configuraciones directamente en el sistema operativo.

En Azure, Docker permite aislar el backend y PostgreSQL, facilitando su despliegue y administración. Como desventaja, requiere configurar imágenes, volúmenes, redes y contenedores.

**2. ¿Quién le pide los datos a la API: el servidor de EC2 o el navegador del usuario? ¿Cómo lo comprobaron?**

El navegador del usuario realiza directamente las solicitudes a la API de Azure. Esto se puede comprobar mediante las herramientas de desarrollo del navegador, en la pestaña Network, donde aparece la petición al endpoint `/api/sistemas`.

**3. ¿Qué es CORS y qué tuvieron que configurar para que funcionara?**

CORS es un mecanismo de seguridad de los navegadores que controla las solicitudes entre diferentes orígenes. Para permitir la comunicación se configura `ORIGEN_PERMITIDO` en el backend con la dirección HTTPS exacta del frontend.

**4. ¿Por qué la base de datos no debe tener un puerto abierto a Internet?**

Porque permitiría que cualquier persona intentara conectarse directamente a PostgreSQL. Al mantenerla dentro de la red privada de Docker Compose, solamente los servicios autorizados pueden comunicarse con ella.

## 21. Información de despliegue

### Backend Azure

- Dominio: https://api-anyu.scaleautomatization.tech
- IP pública: 172.149.24.44
- Imagen Docker: `andrewbar1244/so-backend:1.0`
- Base de datos: PostgreSQL 16
- Servidor web: Nginx
- Certificados: Let's Encrypt mediante Certbot

### Frontend AWS

- Dominio: pendiente de confirmación.
- IP pública: pendiente de registrar.
- Servidor web: Nginx.
- Sistema operativo: Ubuntu Server 24.04.

### Repositorio GitHub

https://github.com/oteroyulian461-alt/proyecto-SO

### Docker Hub

https://hub.docker.com/r/andrewbar1244/so-backend

## 22. Costos y limpieza

El proyecto utiliza los servicios gratuitos o créditos educativos disponibles de AWS, Azure, Docker Hub y GitHub, de acuerdo con las condiciones de cada plataforma.

Se recomienda:

- Configurar una alerta de presupuesto en cero en AWS.
- Revisar los créditos disponibles de Azure for Students.
- Utilizar tamaños pequeños de máquinas virtuales.
- No construir imágenes Docker directamente en las máquinas virtuales.
- Mantener protegidas las credenciales.
- Después de recibir la calificación, terminar la instancia EC2 y eliminar el grupo de recursos de Azure, según las instrucciones del docente.

---

**Fin de la documentación del proyecto.**