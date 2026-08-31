# 🚀 Pipeline completo integrando prácticas de seguridad y monitoreo

Pipeline de despliegue automático de una aplicación Node.js/Express hacia Google Cloud Run, usando Workload Identity Federation para autenticación sin claves secretas.


---

## 1. 🏗️ Introducción y Arquitectura del Sistema

Este proyecto implementa un flujo de integración y despliegue continuo (CI/CD) para una aplicación web desarrollada con Node.js y Express. El proceso utiliza GitHub Actions como pipeline principal para automatizar el análisis de código, la construcción y publicación de imágenes Docker y el despliegue de la aplicación en Google Cloud Run.

Además, se configuró Jenkins como pipeline adicional, SonarQube Cloud para el análisis de calidad y seguridad del código, y Google Cloud Monitoring para visualizar métricas y generar alertas sobre el comportamiento de la aplicación en producción.

```
git push (main)
      │
      ▼
GitHub Actions (ubuntu-latest)
      │
      ├─ 1. Checkout código
      ├─ 2. SonarQube Scan
      ├─ 3. Auth → Google Cloud (Workload Identity OIDC)
      ├─ 4. Login Docker → Artifact Registry
      ├─ 5. docker build & push (taggeado con commit SHA)
      └─ 6. gcloud run deploy → Cloud Run
              │ ------------------------                        
              ▼                         ▼
    Google Cloud Monitoring      Servicio en vivo (HTTPS)


GitHub
      │
      ▼
Jenkins (Jenkinsfile)
      │
      ├─ Test Jenkins
      ├─ Test Docker
      └─ Test Google Cloud
```
---

## 2. Stack:
- **Aplicación:** Node.js + Express
- **Contenedor:** Docker
- **Registro de imágenes:** Google Artifact Registry
- **Plataforma:** Google Cloud Run (serverless)
- **CI/CD:** GitHub Actions + Jenkins
- **Análisis de código:** SonarQube Cloud
- **Monitoreo:** Google Cloud Monitoring
- **Métricas:** SLIs de errores 5xx, latencia p95, disponibilidad e instancias
- **Alertas:** Google Cloud Monitoring → notificaciones por correo
- **Autenticación:** Workload Identity Federation (OIDC, sin claves JSON)

---
## 3. ⚙️ Pipeline de Integración y Despliegue Continuo (GitHub Actions)
El pipeline se encuentra configurado en la ruta `.github/workflows/deploy.yml` y se ejecuta de manera automática ante cada evento de `push` a la rama principal.
### Código de Configuración (`deploy.yml`)
```yaml
name: Deploy to Cloud Run

on:
  push:
    branches: [main]


env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  REGION: us-central1
  SERVICE_NAME: mi-app
  IMAGE: us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/mi-app/mi-app

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - name: Checkout código
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@7006c4492b2e0ee0f816d36501671557c97f5995
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: https://sonarcloud.io

      - name: Autenticar con Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
          service_account: ${{ secrets.WIF_SERVICE_ACCOUNT }}
      - name: Configurar gcloud
        uses: google-github-actions/setup-gcloud@v2
        
      - name: Configurar Docker para Artifact Registry
        run: gcloud auth configure-docker us-central1-docker.pkg.dev

      - name: Build y push imagen Docker
        run: |
          docker build -t $IMAGE:${{ github.sha }} .
          docker push $IMAGE:${{ github.sha }}

      - name: Deploy a Cloud Run
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: ${{ env.SERVICE_NAME }}
          region: ${{ env.REGION }}
          image: ${{ env.IMAGE }}:${{ github.sha }}
```
---
## 4. ⚙️ Jenkins 
Jenkins se configuró como un pipeline independiente para validar la ejecución de Jenkins, Docker y la disponibilidad de Google Cloud desde el servidor Jenkins.
### Código de Configuración (`Jenkinsfile`)
```Jenkinsfile
pipeline {
    agent any

    stages {
        stage('Test Jenkins') {
            steps {
                echo 'Jenkins está ejecutando correctamente el pipeline'
            }
        }

        stage('Test Docker') {
            steps {
                bat 'docker --version'
            }
        }

        stage('Test Google Cloud') {
            steps {
                bat 'gcloud --version'
            }
        }
    }
}
```
---
## 5. 📊 Evidencias de Ejecución y Funcionamiento
### A. 🟢 Ejecución Exitosa del Pipeline en GitHub Actions
El ultimo commit que se realizó fue "Integrate SonarQube Cloud" y en el modulo de Actions de github nos da como resultado exitoso. 
<img width="931" height="731" alt="image" src="https://github.com/user-attachments/assets/568d0573-2e49-43c3-80b2-557c9b805982" />

También es importante tener en cuenta que se implementaron 4 secretas en github incluyendo la de sonar. 
<img width="661" height="300" alt="image" src="https://github.com/user-attachments/assets/204f5503-11d5-4e75-9780-14017cb3da21" />

### B. 🟢 Ejecución Exitosa de Jenkins
Jenkins se configuró como un pipeline adicional para demostrar la ejecución automatizada de procesos de integración continua sobre el mismo repositorio de GitHub. A diferencia de GitHub Actions, que actualmente realiza el despliegue completo de la aplicación en Google Cloud Run, Jenkins se utiliza para validar la disponibilidad y correcta integración de las principales herramientas del entorno.

* **Resultado:** Se evidencia que el resultado es exitoso. 
<img width="1877" height="513" alt="image" src="https://github.com/user-attachments/assets/c855ee6d-b70c-4b0d-99a8-cf9d208ebfa5" />
* **Consola:**

```Consola de Jenkins
Lanzada por el usuario Edgar Andres Romero Otalora
Obtained Jenkinsfile from git https://github.com/edgarrootalora/cicdtraining.git
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins  in C:\ProgramData\Jenkins\.jenkins\workspace\cicdtraining-cd
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
 > git.exe rev-parse --resolve-git-dir C:\ProgramData\Jenkins\.jenkins\workspace\cicdtraining-cd\.git # timeout=10
Fetching changes from the remote Git repository
 > git.exe config remote.origin.url https://github.com/edgarrootalora/cicdtraining.git # timeout=10
Fetching upstream changes from https://github.com/edgarrootalora/cicdtraining.git
 > git.exe --version # timeout=10
 > git --version # 'git version 2.54.0.windows.1'
 > git.exe fetch --tags --force --progress -- https://github.com/edgarrootalora/cicdtraining.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git.exe rev-parse "refs/remotes/origin/main^{commit}" # timeout=10
Checking out Revision a3a24f26b9632c89906ecccce00882ad48a5ee4b (refs/remotes/origin/main)
 > git.exe config core.sparsecheckout # timeout=10
 > git.exe checkout -f a3a24f26b9632c89906ecccce00882ad48a5ee4b # timeout=10
Commit message: "Integrate SonarQube Cloud"
 > git.exe rev-list --no-walk 4839f9664f7f70b403a0e25cb88bc6aadc28c23d # timeout=10
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Test Jenkins)
[Pipeline] echo
Jenkins está ejecutando correctamente el pipeline
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Test Docker)
[Pipeline] bat

C:\ProgramData\Jenkins\.jenkins\workspace\cicdtraining-cd>docker --version 
Docker version 29.7.2, build a7dcaa6
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Test Google Cloud)
[Pipeline] bat

C:\ProgramData\Jenkins\.jenkins\workspace\cicdtraining-cd>gcloud --version 
Google Cloud SDK 582.0.0
alpha 2026.08.21
beta 2026.08.21
bq 2.1.37
core 2026.08.21
gcloud-crc32c 1.0.0
gsutil 5.37
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```
  
---

### C. 🟢 Implementación de SonarQube Cloud
Se integró SonarQube Cloud al pipeline de GitHub Actions con el objetivo de analizar automáticamente la calidad y seguridad del código fuente cada vez que se realiza un push sobre la rama main.
* **Ejecución:** La ejecución de sonar en el pipeline fue exitosa.

* **Resultados:** En las imágenes se ven los resultados arrojados. El análisis también generó recomendaciones relacionadas con seguridad, entre ellas el uso de versiones específicas para dependencias, ejecución de contenedores con usuario root, uso de scripts durante la instalación de paquetes y otras prácticas de seguridad.

Estos resultados permiten evidenciar que SonarQube no solamente fue integrado al pipeline, sino que realizó un análisis real del código y proporcionó información para identificar posibles mejoras.

<img width="1467" height="889" alt="image" src="https://github.com/user-attachments/assets/c644c9d7-02ac-489c-bafb-b626b2baa2e3" />

<img width="1134" height="617" alt="image" src="https://github.com/user-attachments/assets/699bf4b4-84fe-4e71-b773-14d543ee4e38" />

<img width="1689" height="831" alt="image" src="https://github.com/user-attachments/assets/ebcd6778-a982-44e6-a92f-83cc50117eb4" />


### D. 🟢 SLIs
Mediante el archivo dashboard.json se definió la estructura del dashboard utilizado para medir y visualizar los principales indicadores de nivel de servicio (SLI) de la aplicación.

* **Panel general de los SLIs** En la imagen se pueden observar las 4 gráficas donde se analiza: La tasa de errores 5xx, la Latencia p95(ms), Disponibilidad (request 2xx vs total) y Concurrencia - Requests activos
<img width="1692" height="794" alt="image" src="https://github.com/user-attachments/assets/c30732f5-1567-43f2-8b27-11c64aee8e7c" />

* **Tasa de errores** mide la frecuencia con la que el servicio responde con errores internos del servidor (códigos HTTP 5xx). Permite identificar fallos en la aplicación y evaluar su confiabilidad.
  <img width="760" height="565" alt="image" src="https://github.com/user-attachments/assets/f978756b-6b39-4a18-8c4d-21785089ee20" />

* **Latencia** mide el tiempo de respuesta del 95% de las solicitudes. Permite identificar si la aplicación está respondiendo rápidamente y detectar posibles problemas de rendimiento.
  <img width="758" height="560" alt="image" src="https://github.com/user-attachments/assets/ab9945fb-3ec6-4759-8715-bcc7a66a13b0" />
  
* **Disponibilidad** mide la proporción de solicitudes que reciben una respuesta exitosa (códigos HTTP 2xx) frente al total de solicitudes.
<img width="756" height="558" alt="image" src="https://github.com/user-attachments/assets/acc535a1-56a4-4a18-be4f-bd3c4101a9f9" />

* **Concurrencia** mide la cantidad de instancias de la aplicación que están atendiendo solicitudes simultáneamente. Permite conocer el nivel de carga y cómo escala el servicio ante el tráfico.
<img width="755" height="563" alt="image" src="https://github.com/user-attachments/assets/09a9e95a-c44d-4514-9816-558900c701f8" />

### E. 🟢 Alertas
Se implementaron 3 alertas para indicar cuando se presenten problemas de Tasa de errores, latencia y disponibilidad en google cloud. 

<img width="1149" height="291" alt="image" src="https://github.com/user-attachments/assets/22f07246-23a5-4948-a723-4c8f45ef47ed" />

También se implementaron dos canales de notificación vía correo electrónico, configurados para recibir las alertas generadas por Google Cloud Monitoring cuando alguna de las métricas definidas supera los umbrales establecidos. De esta manera, se facilita la detección oportuna de posibles problemas en el funcionamiento y rendimiento de la aplicación.

### F. Prueba de carga con K6
Se realizo una prueba de carga con K6 que simula hasta 200 usuarios concurrentes accediendo a la aplicación Cloud Run. Esto mediante el archivo load-test.js que se encuentra en el directorio K6. 

#### Umbrales definidos

Para evaluar el comportamiento de la aplicación se establecieron los siguientes SLO:

- **Tasa de errores:** menor al 0.5%.
- **Latencia p95:** menor a 500 ms.
- **Tasa de errores personalizada:** menor al 0.5%.

Estos umbrales permiten determinar automáticamente si la aplicación cumple con los niveles de servicio esperados durante la prueba de carga.

Al ejecutarlo muestra el siguiente resultado: 

<img width="311" height="245" alt="image" src="https://github.com/user-attachments/assets/0f96d36e-42fd-447d-90c1-50d042963c9b" />

La prueba generó tráfico normal, tráfico lento y errores simulados. La tasa de errores se mantuvo dentro del umbral establecido, mientras que la latencia superó el límite definido y la disponibilidad alcanzó un 95.2%.

### G. Emails resultado de la prueba de carga
Después de la prueba de cargar recibimos correos electrónicos relacionados a las alertas de SLIs. 

* **Alerta de tasa de error disparada** A continuación la alerta de tasa de error
<img width="1393" height="715" alt="image" src="https://github.com/user-attachments/assets/fa3c5d5c-7259-48bb-b95d-2dfade31362f" />

* **Alerta de tasa de error recuperada**
<img width="1099" height="689" alt="image" src="https://github.com/user-attachments/assets/b0a1ad32-78d9-4b55-ba5d-08910b3feebf" />

* **Alerta de latencia disparada**
<img width="1404" height="701" alt="image" src="https://github.com/user-attachments/assets/a9bf6773-a9f0-4134-8d01-799bb207e13f" />

* **Alerta de latencia recuperada**
<img width="1167" height="634" alt="image" src="https://github.com/user-attachments/assets/48aee159-63e9-42fc-8251-e0c49f61bf83" />

* **Alerta de disponibilidad disparada**
<img width="1049" height="759" alt="image" src="https://github.com/user-attachments/assets/f3dc3b3b-f067-4409-8273-d4907d1f11c3" />

* **Alerta de disponibilidad recuperada**
<img width="1061" height="744" alt="image" src="https://github.com/user-attachments/assets/ecaa9009-6663-4a3b-83fc-7ede62473729" />

## 6. 📝 Conclusiones

La implementación permitió construir un flujo CI/CD para una aplicación Node.js/Express utilizando GitHub Actions como pipeline principal de despliegue. Se integró SonarQube Cloud para el análisis de calidad y seguridad del código y Jenkins como pipeline adicional para validar la integración de las herramientas utilizadas.

La aplicación fue contenerizada mediante Docker, publicada en Google Artifact Registry y desplegada en Google Cloud Run utilizando Workload Identity Federation, evitando el uso de claves JSON.

Para el monitoreo se utilizaron las capacidades de Google Cloud Monitoring, mediante la definición de SLIs relacionados con errores, latencia, disponibilidad y concurrencia. Adicionalmente, se configuraron alertas mediante correo electrónico y se realizó una prueba de carga con K6 para evaluar el comportamiento de la aplicación bajo diferentes niveles de tráfico.

Los resultados de la prueba permitieron comprobar el funcionamiento del sistema de monitoreo, incluyendo la generación y recuperación de alertas cuando las métricas superaron los umbrales establecidos.




