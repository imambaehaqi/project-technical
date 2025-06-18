```markdown
# Technical Assessment: Mobile & Backend Application with CI/CD

Ini adalah proyek Technical Assessment yang mendemonstrasikan pengembangan aplikasi mobile (React Native) dan backend (Spring Boot) dengan implementasi Continuous Integration/Continuous Deployment (CI/CD) dan deployment ke Kubernetes.

## 1. Deskripsi Proyek

Proyek ini terdiri dari:
* **Backend:** Aplikasi RESTful API sederhana menggunakan Spring Boot (Java).
* **Mobile:** Aplikasi mobile sederhana menggunakan React Native, yang mengonsumsi API dari backend.

Deployment diotomatisasi menggunakan prinsip CI/CD dengan Jenkins Pipeline, dan aplikasi backend di-containerize dengan Docker lalu di-deploy ke Kubernetes.

## 2. Struktur Proyek

project-technical-assesment/
├── backend/                  # Kode Spring Boot, Dockerfile, Jenkinsfile Backend
├── mobile/                   # Kode React Native, Jenkinsfile Mobile
├── kubernetes/               # File konfigurasi Kubernetes (YAMLs)
├── README.md                 # Dokumentasi proyek ini
└── .gitignore

## 3. Desain CI/CD Flow (Poin 2 Assessment)

Alur CI/CD dirancang untuk otomatisasi penuh dari perubahan kode hingga deployment. Jenkins Pipeline mengorkestrasi proses ini.

### Alur Umum:
1.  **Code Commit:** Developer mendorong perubahan kode ke Git.
2.  **Jenkins Trigger:** Pipeline Jenkins terpicu secara otomatis.
3.  **CI (Continuous Integration):**
    * Checkout kode.
    * Build aplikasi (Maven untuk Backend, npm/Gradle untuk Mobile).
    * Jalankan unit tests.
    * (Backend saja) Build Docker Image.
    * (Backend saja) Push Docker Image ke Docker Registry.
4.  **CD (Continuous Deployment/Delivery):**
    * (Backend saja) Deploy ke Kubernetes menggunakan manifest YAML.
    * (Mobile saja) Archive APK sebagai artifact.

## 4. Implementasi

### 4.1. Backend (Spring Boot)
* **Source Code:** Aplikasi Spring Boot sederhana dengan endpoint `/` dan `/hello`.
* **Dockerfile (Poin 1 Assessment):**
    ```dockerfile
    FROM openjdk:17-jdk-slim
    ARG JAR_FILE=target/springboot-backend-app-0.0.1-SNAPSHOT.jar
    COPY ${JAR_FILE} app.jar
    EXPOSE 8080
    ENTRYPOINT ["java", "-jar", "/app.jar"]
    ```
* **Jenkinsfile (Poin 3 Assessment):**
    ```groovy
    pipeline {
        agent any
        environment {
            DOCKER_REGISTRY = 'your-docker-registry.com' // GANTI
            DOCKER_IMAGE_NAME = 'springboot-backend-app'
            DOCKER_IMAGE_TAG = "${env.BUILD_NUMBER}"
            KUBERNETES_NAMESPACE = 'default'
            DOCKER_CREDENTIALS_ID = 'your-docker-registry-credentials-id' // GANTI
            KUBERNETES_MANIFESTS_DIR = 'kubernetes'
        }
        stages {
            stage('Checkout Source Code') { steps { git branch: 'main', url: '[https://github.com/your-username/your-repo.git](https://github.com/your-username/your-repo.git)' }} // GANTI
            stage('Build Spring Boot Application') { steps { sh 'mvn clean package -DskipTests' }}
            stage('Build Docker Image') { steps { script { docker.build "${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}", '.' }}}
            stage('Push Docker Image to Registry') { steps { script { docker.withRegistry("https://${DOCKER_REGISTRY}", DOCKER_CREDENTIALS_ID) { docker.image("${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}").push() }}}}
            stage('Deploy to Kubernetes') {
                steps {
                    script {
                        sh "sed -i 's|image: springboot-backend-image:latest|image: ${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}|g' ${KUBERNETES_MANIFESTS_DIR}/backend-deployment.yaml"
                        sh "sed -i 's|imagePullPolicy: Never|imagePullPolicy: Always|g' ${KUBERNETES_MANIFESTS_DIR}/backend-deployment.yaml"
                        sh "kubectl apply -f ${KUBERNETES_MANIFESTS_DIR}/backend-deployment.yaml --namespace=${KUBERNETES_NAMESPACE}"
                        sh "kubectl apply -f ${KUBERNETES_MANIFESTS_DIR}/backend-service.yaml --namespace=${KUBERNETES_NAMESPACE}"
                        sh "kubectl apply -f ${KUBERNETES_MANIFESTS_DIR}/ingress.yaml --namespace=${KUBERNETES_NAMESPACE}"
                    }
                }
            }
        }
        post { always { cleanWs() } success { echo "Pipeline Backend berhasil!" } failure { echo "Pipeline Backend gagal!" }}
    }
    ```

### 4.2. Mobile (React Native)
* **Source Code:** Aplikasi React Native dengan `axios` untuk memanggil API backend.
    ```javascript
    // Konten App.js yang relevan
    // Perhatikan URL berikut untuk testing lokal:
    // const url = '[http://192.168.58.2/hello](http://192.168.58.2/hello)'; // GANTI dengan IP Minikube Anda yang sebenarnya
    ```
* **Jenkinsfile (Poin 3 Assessment):**
    ```groovy
    pipeline {
        agent any
        environment {
            ANDROID_SDK_ROOT = "/opt/android-sdk"
            PATH = "${tool 'Node_18.x_LTS'}/bin:${env.ANDROID_SDK_ROOT}/platform-tools:${env.ANDROID_SDK_ROOT}/cmdline-tools/latest/bin:${env.ANDROID_SDK_ROOT}/build-tools/34.0.0:${env.PATH}"
        }
        tools { nodejs 'Node_18.x_LTS' }
        stages {
            stage('Checkout Source Code') { steps { git branch: 'main', url: '[https://github.com/your-username/your-repo.git](https://github.com/your-username/your-repo.git)' }} // GANTI
            stage('Install Node.js Dependencies') { steps { sh 'npm install' }}
            stage('Run Lint & Tests') { steps { sh 'npm run lint'; sh 'npm test' }}
            stage('Build Android App') { steps { sh 'chmod +x android/gradlew'; dir('android') { sh './gradlew clean assembleRelease' }}}
            stage('Archive Artifacts') { steps { archiveArtifacts artifacts: 'android/app/build/outputs/apk/release/**/*.apk', fingerprint: true }}
            // stage('Distribute App') { ... } // Opsional: Untuk distribusi Firebase/Play Store
        }
        post { always { cleanWs() } success { echo "Pipeline Mobile berhasil!" } failure { echo "Pipeline Mobile gagal!" }}
    }
    ```

## 5. Deployment ke Kubernetes (Poin 4 Assessment)

Aplikasi backend di-deploy ke Minikube (cluster Kubernetes lokal) menggunakan manifest YAML.

* **`backend-deployment.yaml`:** Mengelola Pod backend (2 replika), image `springboot-backend-image:latest` dengan `imagePullPolicy: Never` untuk testing lokal.
    ```yaml
    # Isi backend-deployment.yaml Anda di sini (sesuai yang terakhir kita sepakati)
    # Penting: image: springboot-backend-image:latest dan imagePullPolicy: Never
    ```
* **`backend-service.yaml`:** Menyediakan akses internal (`ClusterIP`) ke Pod backend pada port 80.
    ```yaml
    # Isi backend-service.yaml Anda di sini
    ```
* **`ingress.yaml`:** Mengekspos backend ke luar cluster melalui `api.yourdomain.com` (port 80) menggunakan Nginx Ingress Controller.
    ```yaml
    # Isi ingress.yaml Anda di sini
    # Penting: host: api.yourdomain.com
    ```

## 6. Cara Menjalankan Proyek (Lokal)

### 6.1. Prasyarat
Pastikan Anda telah menginstal: Docker, Minikube, `kubectl`, JDK 17+, Maven, Node.js (LTS), npm, Android Studio (SDK & emulator), dan Expo Go di HP/emulator Anda.

### 6.2. Setup & Akses Backend di Minikube
1.  **Mulai Minikube & Ingress:** `minikube start --driver=docker` lalu `minikube addons enable ingress`.
2.  **Set Docker Env:** `eval $(minikube docker-env)`.
3.  **Build Backend & Docker Image:** Di `backend/`, `mvn clean package -DskipTests` lalu `docker build -t springboot-backend-image:latest .`.
4.  **Apply K8s Manifests:** Di root proyek, `minikube kubectl -- apply -f kubernetes/backend-deployment.yaml -f kubernetes/backend-service.yaml -f kubernetes/ingress.yaml`.
5.  **Dapatkan Minikube IP:** `minikube ip`. Catat IP-nya (misal: `192.168.58.2`).
6.  **Edit File `hosts`:** Tambahkan baris `<MINIKUBE_IP_ANDA> api.yourdomain.com` ke file `hosts` sistem Anda.
7.  **Verifikasi Browser:** Akses `http://api.yourdomain.com/hello`.

### 6.3. Setup & Akses Mobile (React Native)
1.  **Instal Dependensi:** Di `mobile/`, `npm install`.
2.  **Jalankan Expo Go:** `npx expo start`. Pindai QR dari HP atau jalankan di emulator.
3.  **Uji Koneksi:** Aplikasi akan mencoba `http://192.168.58.2/hello`. Pastikan terkoneksi.
    * **Catatan Penting:** Jika ada "Network request failed" dari HP/emulator, kemungkinan besar ini adalah masalah *firewall* di host Anda (`ufw` di Ubuntu). Anda mungkin perlu menonaktifkan UFW sementara (`sudo ufw disable`) atau menambahkan aturan yang mengizinkan lalu lintas masuk ke IP Minikube pada port 80 dari jaringan Anda (misal: `sudo ufw allow in on br-7bbd69e724f4 to any port 80 proto tcp`).

 ---