### 1. Gestión de Código fuente (Git)

Si el ejercicio te pide inicializar un repositorio, guardar cambios o simular trabajo en equipo.

* **¿Qué debes tener?** Un repositorio local y/o remoto. No requiere archivos especiales más allá de tu código y un `.gitignore` para excluir archivos innecesarios.
* **Comandos clave:**
* `git init`: Inicializa el repositorio.
* `git add .`: Agrega todos los archivos modificados al área de preparación (staging).
* `git commit -m "Mensaje descriptivo"`: Guarda los cambios localmente.
* `git push origin main`: Sube los cambios al repositorio remoto.
* `git pull`: Trae y fusiona los cambios del repositorio remoto.



### 2. Contenedorización (Docker)

Si el requerimiento es empaquetar una aplicación para que funcione en cualquier entorno o crear una imagen.

* **¿Qué archivo debes crear?** Un archivo llamado exactamente `Dockerfile` (sin extensión) en la raíz del proyecto.
* **Estructura básica del `Dockerfile`:**
```dockerfile
FROM node:18-alpine # 1. Imagen base (Node, Python, Nginx, etc.)
WORKDIR /app        # 2. Directorio de trabajo dentro del contenedor
COPY . .            # 3. Copiar archivos locales al contenedor
RUN npm install     # 4. Instalar dependencias
EXPOSE 8080         # 5. Exponer el puerto
CMD ["npm", "start"]# 6. Comando para iniciar la app

```


* **Comandos clave:**
* `docker build -t mi-app:v1 .`: Construye la imagen y le asigna un nombre y etiqueta (tag). El punto `.` indica el directorio actual.
* `docker run -d -p 8080:80 mi-app:v1`: Ejecuta el contenedor en segundo plano (`-d`) y mapea el puerto 8080 de tu máquina al 80 del contenedor.
* `docker ps`: Lista los contenedores en ejecución.
* `docker stop <id_contenedor>`: Detiene un contenedor.



### 3. Integración y Despliegue Continuo (CI/CD)

Si te piden automatizar pruebas o despliegues al hacer un "push" al repositorio (ej. usando GitHub Actions).

* **¿Qué archivo debes crear?** Un archivo YAML (ej. `deploy.yml`) dentro de la ruta `.github/workflows/`.
* **Estructura básica del YAML:**
```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [ "main" ] # Se ejecuta al hacer push a main
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout del código
      uses: actions/checkout@v3
    - name: Construir imagen Docker
      run: docker build -t mi-app:latest .
    # (Aquí irían los pasos para subir la imagen y actualizar Kubernetes)

```



### 4. Orquestación y Despliegue (Kubernetes)

Si el escenario involucra escalar la aplicación, asegurar alta disponibilidad o aplicar una estrategia de despliegue específica (como Blue-Green o Rolling Update).

* **¿Qué archivos debes crear?** Manifiestos en formato YAML. Por lo general, necesitarás un `deployment.yaml` y un `service.yaml`.
* **Estructura básica de un Deployment (ej. `deployment.yaml`):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app-deployment
spec:
  replicas: 3 # Número de instancias
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
        version: "v1" # Clave para estrategias Blue-Green
    spec:
      containers:
      - name: mi-app-container
        image: mi-app:v1
        ports:
        - containerPort: 80

```


* **Estructura básica de un Service (ej. `service.yaml`):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mi-app-service
spec:
  selector:
    app: mi-app # Debe coincidir con el label del Deployment
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer

```


* **Comandos clave:**
* `kubectl apply -f archivo.yaml`: Crea o actualiza el recurso definido en el YAML.
* `kubectl get pods`: Lista todos los pods y su estado.
* `kubectl get svc`: Lista los servicios y sus direcciones IP.
* `kubectl set image deployment/mi-app-deployment mi-app-container=mi-app:v2`: Actualiza la imagen de un deployment en caliente.



### 5. Resolución de Problemas (Troubleshooting)

Muy probablemente haya una parte del examen donde algo "esté roto" o no inicie correctamente.

* **Comandos de diagnóstico:**
* `docker logs <id_contenedor>`: Revisa qué falló dentro del contenedor.
* `kubectl describe pod <nombre_del_pod>`: Muestra los eventos del pod (útil si hay errores de falta de memoria, imagen no encontrada, o problemas de persistencia de almacenamiento).
* `kubectl logs <nombre_del_pod>`: Muestra los logs de la aplicación corriendo dentro de Kubernetes.
* `kubectl port-forward svc/mi-app-service 8080:80`: Útil para probar localmente un servicio que está dentro del clúster antes de exponerlo públicamente.



---

UNIDAD 3 Y 4 PRACTICA

### 1. Archivos Necesarios a Crear

Para lograr que dos procesos corran en un solo contenedor, necesitas un archivo que actúe como gestor de procesos. El enfoque más rápido para el examen es crear un script de inicio (`start.sh`) y el `Dockerfile`.

**Archivo 1: `start.sh` (El gestor de procesos)**
Crea este archivo en la raíz del proyecto. Su función es levantar PostgreSQL en segundo plano, crear la base de datos (si no existe) y luego ejecutar la aplicación de Spring Boot en primer plano para que el contenedor no se detenga.

```bash
#!/bin/bash
# 1. Iniciar el servicio de PostgreSQL
service postgresql start

# 2. Configurar el usuario y base de datos (ajusta credenciales según el application.properties)
su - postgres -c "psql -c \"CREATE USER admin WITH PASSWORD 'admin';\"" || true
su - postgres -c "psql -c \"CREATE DATABASE inventario OWNER admin;\"" || true

# 3. Iniciar la aplicación Spring Boot
java -jar /app/target/inventario-app.jar

```

**Archivo 2: `Dockerfile**`
Debe compilar la aplicación, instalar Java y PostgreSQL, y ejecutar el script.

```dockerfile
# ETAPA DE CONSTRUCCIÓN (Build)
FROM maven:3.8.5-openjdk-17 AS builder
WORKDIR /app
# Copiar primero el pom.xml para aprovechar el caché
COPY pom.xml .
RUN mvn dependency:go-offline
# Copiar el código fuente
COPY src ./src
RUN mvn clean package -DskipTests

# ETAPA FINAL
FROM ubuntu:22.04
WORKDIR /app

# Instalar Java 17 y PostgreSQL
RUN apt-get update && \
    apt-get install -y openjdk-17-jdk postgresql postgresql-contrib curl && \
    apt-get clean

# Copiar el .jar desde la etapa de construcción
COPY --from=builder /app/target/*.jar /app/target/inventario-app.jar

# Copiar y dar permisos al script de inicio
COPY start.sh /app/start.sh
RUN chmod +x /app/start.sh

# Exponer el puerto de la app (5000 según las properties)
EXPOSE 5000

# Ejecutar el script que levanta ambos servicios
CMD ["/app/start.sh"]

```

### 2. Comandos de Ejecución (Para las capturas de pantalla)

1. **Construir la imagen:**
`docker build -t tuusuario/inventario-api:v1 .`
2. **Publicar en Docker Hub:**
`docker login`
`docker push tuusuario/inventario-api:v1`
3. **Ejecutar con persistencia de datos (Crucial para los 3.0 puntos):**
`docker run -d -p 5000:5000 -v inventario_data:/var/lib/postgresql/14/main --name api-test tuusuario/inventario-api:v1`
*(Nota: El flag `-v` asegura que si detienes o eliminas el contenedor `api-test`, la información de los productos siga ahí cuando levantes uno nuevo usando ese mismo volumen `inventario_data`).*

### 3. Justificación Técnica (Entregable 2)

Estas son las respuestas estructuradas que debes colocar en tu reporte para asegurar el puntaje de diseño técnico:

* **Imagen base:** "Se descartó el uso de imágenes ligeras como Alpine porque presentan problemas de compatibilidad con ciertas bibliotecas de PostgreSQL. Se eligió `ubuntu:22.04` como base en la etapa final por su estabilidad para alojar tanto el motor de base de datos como el JRE de Java requerido por Spring Boot."
* **Estrategia de Build:** "Se utilizó un enfoque Multi-stage Build (Etapa de construcción con Maven y Etapa final con Ubuntu). Esto evita que el código fuente y las herramientas de compilación lleguen a la imagen final, reduciendo el peso de la imagen y mejorando la seguridad."
* **Gestión de procesos:** "Dado que un contenedor Docker normalmente gestiona un solo proceso PID 1, se descartó el arranque directo de Java en el CMD. En su lugar, se implementó un script `bash` (`start.sh`) que levanta el demonio de PostgreSQL en background y la aplicación Java en foreground, manteniendo el contenedor vivo."
* **Mecanismo de persistencia:** "Se descartó el uso del sistema de archivos interno del contenedor (efímero). Se configuró un Volumen Nombrado (Named Volume) de Docker mapeado al directorio `/var/lib/postgresql/14/main`, garantizando que la data del sistema de inventario sobreviva a la destrucción del contenedor."

### 4. Respuestas a Preguntas de Análisis (Entregable 4)

* **¿Qué ventaja concreta aporta el orden en que se copian los archivos dentro del Dockerfile? ¿Qué consecuencia tendría invertirlo?**
* *Respuesta:* Copiar primero el `pom.xml` y descargar dependencias aprovecha el sistema de caché de capas (layer caching) de Docker. Las dependencias rara vez cambian, por lo que estas capas se reutilizan en futuros builds. Si se invierte y se copia todo el código fuente primero, cualquier cambio mínimo en un archivo `.java` invalidará la caché y obligará a descargar todas las dependencias de internet nuevamente, haciendo el proceso de CI/CD extremadamente lento.


* **¿Qué información debería publicarse junto a la imagen en Docker Hub para que un tercero pueda usarla sin acceder al código fuente?**
* *Respuesta:* En el archivo `README` de Docker Hub se debe documentar: 1) Los puertos que expone la aplicación (ej. 5000). 2) Las rutas de los volúmenes para la persistencia de datos (ej. `/var/lib/postgresql/...`). 3) Las variables de entorno necesarias para configurar credenciales (si aplicara). 4) Un comando `docker run` de ejemplo listo para copiar y pegar.


* **La aplicación Spring Boot tarda 15 segundos en iniciar; el servidor web, 1 segundo. ¿Qué problema concreto genera esa diferencia y cómo se resuelve?**
* *Respuesta:* El problema es que Docker marcará el contenedor como "En ejecución" (Running) en el instante en que arranca el web server, pero si un usuario intenta consumir la API durante los primeros 14 segundos, recibirá un error (Connection Refused). Se resuelve implementando un `HEALTHCHECK` en el Dockerfile apuntando al endpoint proporcionado `/health`. Así, herramientas como Kubernetes o balanceadores de carga sabrán que no deben enviar tráfico a ese contenedor hasta que el endpoint devuelva un estado HTTP 200 OK.
