# Guía Definitiva: Desarrollo, Docker, Kubernetes y CI/CD desde Cero

Esta guía está diseñada paso a paso para que cualquier persona, incluso sin experiencia previa, pueda tomar un repositorio inicial vacío o con los archivos base, y completar con éxito todos los retos de una práctica o examen técnico de backend, infraestructura y automatización.

---

## Índice General

1. [Fase 0: Configuración Inicial y Reconocimiento del Proyecto (¡Desde Cero!)](https://www.google.com/search?q=%23fase-0-configuraci%C3%B3n-inicial-y-reconocimiento-del-proyecto-desde-cero)
2. [Fase 1: Reto 1 - Aplicación y Pruebas Unitarias Seguras](https://www.google.com/search?q=%23fase-1-reto-1---aplicaci%C3%B3n-y-pruebas-unitarias-seguras)
3. [Fase 2: Reto 2 - Kubernetes (Despliegue y Conectividad con Services)](https://www.google.com/search?q=%23fase-2-reto-2---kubernetes-despliegue-y-conectividad-con-services)
4. [Fase 3: Reto 3 - CI/CD con GitHub Actions](https://www.google.com/search?q=%23fase-3-reto-3---cicd-con-github-actions)
5. [Fase 4: Reto 4 - Operación, Escalabilidad y Cero Interrupciones](https://www.google.com/search?q=%23fase-4-reto-4---operaci%C3%B3n-escalabilidad-y-cero-interrupciones)

---

## Fase 0: Configuración Inicial y Reconocimiento del Proyecto (¡Desde Cero!)

Cuando te entregan la práctica o el repositorio por primera vez, tienes una carpeta comprimida o un repositorio de GitHub. Sigue estos pasos exactos antes de tocar cualquier código:

### 1. Abrir el proyecto en tu editor

* Abre **Visual Studio Code (VS Code)**.
* Ve a `File -> Open Folder...` y selecciona la carpeta del proyecto (por ejemplo, `app-ejemplo-evaluacion`).
* Abre la terminal integrada en VS Code presionando las teclas `Ctrl + `` (o ve arriba a `Terminal -> New Terminal`).

### 2. Reconocer los archivos principales

Verifica qué archivos te han dado en la raíz del proyecto. Generalmente encontrarás:

* **`server.js`** (o `app.js`): El código del servidor backend en Node.js.
* **`package.json`**: El archivo que lista las dependencias y los scripts del proyecto (como `npm start` o `npm test`).
* **`Dockerfile`**: Las instrucciones para empaquetar tu aplicación en un contenedor Docker.
* **Carpeta `test/**`: Contiene las pruebas unitarias (ej. `server.test.js`).
* **Carpeta `k8s/` o archivo `deployment.yaml**`: Los manifiestos de Kubernetes.

### 3. Instalar dependencias locales

Para que Node.js reconozca las librerías necesarias del proyecto, ejecuta en tu terminal:

```bash
npm install

```

### 4. Probar la aplicación de forma local

Antes de dockerizar o subir nada, asegúrate de que el servidor enciende bien:

```bash
node server.js

```

*(O si el `package.json` tiene un script de inicio, usa `npm start`).*
Abre tu navegador web e ingresa a `http://localhost:8080`. Deberías ver un mensaje JSON con la versión y el saludo. Presiona `Ctrl + C` en la terminal para apagarlo una vez verificado.

---

## Fase 1: Reto 1 - Aplicación y Pruebas Unitarias Seguras

### 1. Construcción y Ejecución local en Docker

* **Construir la imagen de Docker:**
```bash
docker build -t app-ejemplo-evaluacion .

```


* **Ejecutar el contenedor:**
```bash
docker run -d -p 8080:8080 --name app-prueba app-ejemplo-evaluacion

```


* **Verificar que esté corriendo:**
```bash
docker ps

```



### 2. Configuración de Pruebas Unitarias Seguras (`test/server.test.js`)

*Problema común:* Si una prueba falla, los servidores HTTP mal cerrados se quedan colgados en el puerto y congelan las terminales o los pipelines de CI/CD.
*Solución:* Asegúrate de que el archivo de pruebas use bloques `try...finally` para asegurar el cierre automático de `server.close()`:

```javascript
const { test } = require('node:test');
const assert = require('node:assert');
const http = require('node:http');
const { server } = require('../server.js');

test('GET / responde 200 con un mensaje y una version', async () => {
  await new Promise((resolve) => server.listen(0, resolve));
  const { port } = server.address();

  try {
    const response = await new Promise((resolve, reject) => {
      http.get(`http://localhost:${port}/`, (res) => {
        let data = '';
        res.on('data', (chunk) => (data += chunk));
        res.on('end', () => resolve({ status: res.statusCode, data }));
      }).on('error', reject);
    });

    const parsed = JSON.parse(response.data);
    assert.strictEqual(response.status, 200);
    assert.ok(parsed.message);
    assert.ok(parsed.version);
  } finally {
    server.close(); // Libera el puerto obligatoriamente pase lo que pase
  }
});

```

---

## Fase 2: Reto 2 - Kubernetes (Despliegue y Conectividad con Services)

El fallo más repetitivo en Kubernetes es que el `Service` no encuentre los pods porque las etiquetas (`labels` y `selector`) no coinciden exactamente.

### 1. Crear el Manifiesto Unificado (`k8s/manifest.yaml`)

Crea una carpeta llamada `k8s` y dentro un archivo `manifest.yaml` con este contenido limpio (asegúrate de que `app: webapp` coincida en ambas partes):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: web
          image: app-ejemplo-evaluacion:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 8080

```

### 2. Comandos de Validación Local (Minikube)

* **Iniciar Minikube (si está apagado):**
```bash
minikube start

```


* **Aplicar el manifiesto:**
```bash
kubectl apply -f k8s/manifest.yaml --validate=false

```


* **Verificar que los pods estén corriendo:**
```bash
kubectl get pods

```


* **Verificar que el servicio tenga endpoints activos (evitando que aparezca `<none>`):**
```bash
kubectl get endpoints web-service

```



---

## Fase 3: Reto 3 - CI/CD con GitHub Actions

### 1. Crear el archivo de Workflow (`.github/workflows/ci-cd.yml`)

Crea las carpetas `.github/workflows/` y dentro el archivo `ci-cd.yml`. Este pipeline asegura que el despliegue (`deploy`) **jamás** se ejecute si las pruebas fallan gracias a la directiva `needs`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Instalar dependencias
        run: npm install

      - name: Ejecutar pruebas unitarias
        run: npm test

  deploy:
    needs: build-test
    runs-on: ubuntu-latest
    steps:
      - name: Simular Despliegue Exitoso
        run: echo "Pruebas superadas. Despliegue ejecutado correctamente."

```

---

## Fase 4: Reto 4 - Operación, Escalabilidad y Cero Interrupciones

### 1. Actualizar el Manifiesto para Alta Disponibilidad

Modifica tu archivo `k8s/manifest.yaml` para soportar 3 réplicas, actualizaciones fluidas (`RollingUpdate`) y sondas de salud (`readinessProbe`) que controlan el tráfico de forma inteligente:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: web
          image: app-ejemplo-evaluacion:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 3

```

### 2. Prueba en Vivo de Cero Cortes

1. Aplica los cambios actualizados:
```bash
kubectl apply -f k8s/manifest.yaml --validate=false

```


2. Ejecuta un reinicio controlado para evaluar el comportamiento de los pods:
```bash
kubectl rollout restart deployment/web-deployment

```


3. Monitorea en tiempo real cómo los pods antiguos entran en `Terminating` mientras los nuevos se despliegan de forma escalonada sin interrumpir el servicio:
```bash
kubectl get pods

```
## 💡 Consejos Clave y Buenas Prácticas (Para no fallar en el intento)

### 1. El eterno problema de la imagen en Minikube (ImagePullBackOff)

* **El error:** Cuando aplicas tu manifiesto en Kubernetes, los pods se quedan en estado `ErrImageNeverPull` o `ImagePullBackOff`.
* **La solución:** Recuerda que Minikube corre dentro de su propio entorno aislado (Docker interno). Si construiste la imagen en tu PC principal, Minikube podría no verla a menos que configures el demonio de Docker de Minikube antes de construirla, o uses la política adecuada en el YAML:
```bash
# Si usas Linux/Mac o Docker Desktop con Minikube:
eval $(minikube docker-env)
docker build -t app-ejemplo-evaluacion .

```


Y asegúrate de tener `imagePullPolicy: IfNotPresent` en tu manifiesto.

### 2. Cuidado con los selectores en Kubernetes

* El **90% de los fallos del Reto 2** ocurren porque la etiqueta (`labels: app: ...`) del Deployment no coincide exactamente con el selector (`selector: matchLabels: app: ...`) del Service o del Deployment. Revisa dos veces que las palabras escritas sean idénticas.

### 3. El peligro de los puertos bloqueados en Node.js

* Si ejecutas tus pruebas locales (`npm test`) y la terminal se queda congelada o dice `EADDRINUSE: address already in use`, significa que un servidor anterior se quedó abierto en el puerto 8080.
* **Buenas prácticas:** Utiliza siempre `server.listen(0, ...)` en tus pruebas para que el sistema operativo asigne un puerto aleatorio disponible de forma automática y segura.

### 4. Captura las evidencias *en el momento*

* No dejes las capturas de pantalla para el final. A medida que vayas superando cada reto (cuando veas los tests en verde, los pods en `Running`, o el pipeline exitoso en GitHub Actions), **toma la captura inmediatamente y guárdala en una carpeta llamada `evidencias/**`. Nombrarlas ordenadamente (ej. `reto1-tests.png`, `reto2-endpoints.png`) te ahorrará horas de estrés al armar el PDF final.

### 5. Verifica el escenario negativo del CI/CD

* En los retos de GitHub Actions, los profesores aman evaluar el **escenario negativo** (es decir, que el pipeline se bloquee si subes código con errores). Asegúrate de probar primero una falla intencional (como romper un test o cambiar el código de estado esperado) para capturar el momento exacto en que el trabajo de despliegue (`deploy`) se cancela o se bloquea antes de subir tu versión final y limpia.


---

### 1. Comandos de Emergencia y Diagnóstico en Kubernetes (`kubectl`)

Cuando algo falla en el clúster, estos comandos son tus mejores amigos para saber **exactamente qué pasó**:

* **Ver el historial detallado de un Pod (Ideal para errores de `ImagePullBackOff` o sondas de salud):**
```bash
kubectl describe pod <nombre-del-pod>

```


*(Te muestra las últimas líneas de eventos y si falló la `readinessProbe` o el arranque).*
* **Ver los logs de un Pod en el clúster (Para ver si la app crasheó por código):**
```bash
kubectl logs <nombre-del-pod>

```


* **Ver una vista global de TODO lo que hay en el clúster:**
```bash
kubectl get all

```


*(Te muestra deployments, replicasets, pods y servicios de un solo vistazo).*
* **Hacer un "Rollback" de emergencia (Si una actualización rompió la app):**
```bash
kubectl rollout undo deployment/web-deployment

```


*(Revierte automáticamente al estado anterior que sí funcionaba).*
* **Ver los eventos recientes del clúster ordenados por tiempo:**
```bash
kubectl get events --sort-by='.metadata.creationTimestamp'

```



---

### 2. Comandos de Depuración con Docker

Si la aplicación falla antes de llegar a Kubernetes:

* **Entrar a la terminal interna de un contenedor en ejecución (Para revisar rutas o archivos):**
```bash
docker exec -it <nombre-o-id-del-contenedor> sh

```


*(Una vez dentro, puedes escribir `exit` para salir).*
* **Ver los logs de un contenedor local:**
```bash
docker logs <nombre-o-id-del-contenedor>

```


* **Limpieza profunda de Docker (Si te quedas sin espacio o la caché da problemas):**
```bash
docker system prune -a --volumes

```


*(Borra contenedores detenidos, redes y cachés no utilizadas).*

---

### 3. Solución a Puertos Bloqueados (Muy común en Windows)

Si ejecutas tu app localmente y te sale el error `EADDRINUSE` (puerto en uso):

* **En PowerShell (Windows):**
1. Busca qué proceso está usando el puerto 8080:
```powershell
netstat -ano | findstr :8080

```


*(El número que aparece al final de la línea es el PID).*
2. Mata el proceso de inmediato usando ese PID:
```powershell
taskkill /F /PID <numero_pid>

```