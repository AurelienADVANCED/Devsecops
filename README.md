# Projet E5 DevSecOps – ESTIAM Metz  
## Groupe WMD – Rapport de réalisation  
**Nom : Aurelien ROSELLO**  
**Binôme : Aurian BOHN**  
**Date : 16 avril 2025**

## 📚 Sommaire

1. [🎯 Objectif du projet](#-objectif-du-projet)  
2. [🏗️ Architecture cible](#architecture-cible)
3. [📁 Organisation du projet](#-organisation-du-projet)  
4. [🔧 Étapes de réalisation](#-étapes-de-réalisation)  
5. [🐳 Création des images Docker](#-création-des-images-docker)  
6. [🚀 Optimisation des images Docker](#-optimisation-de-limage-docker-django)  
7. [📦 Déploiement des applications Kubernetes](#-déploiement-des-applications-kubernetes)  
8. [🚀 Commandes de déploiement Kubernetes](#-commandes-de-déploiement-kubernetes)  
9. [✅ Conclusion](#-conclusion)

---

## 🎯 Objectif du projet

L'objectif de ce projet est de concevoir une maquette fonctionnelle d’un cluster Kubernetes local capable d’héberger trois applications web distinctes, dont une application critique développée en Django.  
Chaque application doit être exposée sur un port spécifique (80, 8080 et 9090) et déployée à l’aide de fichiers manifeste Kubernetes indépendants.  
Toutes les images Docker doivent être construites manuellement, optimisées, et intégrées dans un dépôt Git afin d’assurer la traçabilité et la portabilité de l’environnement.

---

<a name="architecture-cible"></a>
## 🏗️ Architecture cible

L’architecture repose sur un cluster Kubernetes local (via Minikube), hébergeant trois applications clonées et adaptées à partir de dépôts publics :

- ✅ **Django Volt** – [django-volt-1744777679](https://github.com/app-generator/django-volt-1744777679)  
- ✅ **Next.js** – [hello-world-next-js](https://github.com/app-generator/hello-world-next-js)  
- ✅ **Flask** – [flask-soft-1744678708](https://github.com/app-generator/flask-soft-1744678708)  

### 🔀 Schéma logique

```
           Navigateur / Client
                    │
   ┌────────────────┼────────────────┐
   │                │                │
   ▼                ▼                ▼
  /django        /front            /flask
  Django          Next.js           Flask
   Port 80       Port 9090         Port 8080

```


## 🔄 Accès aux applications via port-forward (sans Ingress)

Chaque application est exposée en local sur les ports spécifiés dans la consigne :

| Application | Port Kubernetes | Port local | Commande port-forward                              |
|-------------|------------------|--------------------------|-----------------------------------------------------|
| Django      | 5005             | 8081                       | `k port-forward svc/django-service 8081:80`     |
| Flask       | 8080             | 8080                     | `k port-forward svc/flask-service 8080:8080`  |
| Next.js     | 9090             | 9090                     | `k port-forward svc/next-js-service 9090:9090`|

💡 Ces redirections permettent d'accéder directement à :

- `http://localhost:80` → Django  
- `http://localhost:8080` → Flask  
- `http://localhost:9090` → Next.js  

> 🧠 Faire ces commandes dans **3 terminaux séparés**.
---
## 📁 Organisation du projet

![image](https://github.com/user-attachments/assets/87e260df-18c0-4d26-a4f1-fa5b158c9140)

📦 Chaque application dispose de son propre dossier avec son `Dockerfile`.  
📂 Tous les fichiers Kubernetes (`Deployment`, `Service`) sont regroupés dans le dossier `k8s/` pour plus de clarté.

---

## 🔧 Étapes de réalisation

### 🔹 1. Initialiser Minikube

```bash
minikube start
```

## 🐳 Création des images Docker

### 📦 1. Application critique – Django

**Fichier : django-volt/Dockerfile**

```dockerfile
FROM python:3.11-alpine

# Env vars
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Dépendances système
RUN apk update && apk add --no-cache \
    gcc \
    python3-dev \
    musl-dev \
    libffi-dev \
    jpeg-dev \
    zlib-dev \
    postgresql-dev \
    build-base \
    linux-headers \
    libxml2-dev \
    libxslt-dev \
    && pip install --upgrade pip

WORKDIR /app

# Copier uniquement requirements.txt pour tirer parti du cache Docker
COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

# Copier le code de l'app
COPY . .

# Collecte des fichiers statiques & migrations
RUN python manage.py collectstatic --noinput && \
    python manage.py makemigrations && \
    python manage.py migrate

EXPOSE 5005

CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
```

🔐 .dockerignore
```dockerfile
# Répertoires de dev
__pycache__/
*.py[cod]
*.sqlite3
*.log
*.env
*.db
*.bak
*.swp

# Répertoires spécifiques
.env/
venv/
env/
.idea/
.vscode/
.mypy_cache/
.coverage
node_modules/

# Git
.git/
.gitignore

# Fichiers générés par collectstatic
staticfiles/
static/

# Docker
Dockerfile
.dockerignore
```

# Commande de build :
```
docker build -t localhost/django-app:latest .
```

# 📦 2. Application secondaire – Next.js
## Fichier : hello-world-next-js/Dockerfile
```dockerfile
# Étape 1 : Build
FROM node:20-alpine AS builder

WORKDIR /app

COPY package.json ./
RUN npm install

COPY . .
RUN NODE_OPTIONS=--openssl-legacy-provider npm run build

# Étape 2 : Image de production
FROM node:20-alpine

WORKDIR /app

COPY package.json ./
RUN npm install --omit=dev

COPY --from=builder /app/.next .next
COPY --from=builder /app/pages pages
COPY --from=builder /app/node_modules node_modules
COPY --from=builder /app/package.json ./

EXPOSE 9090

CMD ["npx", "next", "start", "-p", "9090"]
```

🔐 .dockerignore
```dockerfile
node_modules
npm-debug.log
.next
Dockerfile
.dockerignore
.git
.gitignore
.vscode
```

## Commande de build :
```
docker build -t localhost/next-js-app:latest .
```

# 3. Application Flask
## Fichier : flask-soft/Dockerfile
```dockerfile
FROM python:3.11-alpine

# Eviter l'écriture de fichiers pyc + affichage live dans le terminal
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Installation des dépendances système
RUN apk update && apk add --no-cache \
    build-base \
    python3-dev \
    libffi-dev \
    musl-dev \
    gcc \
    jpeg-dev \
    zlib-dev \
    postgresql-dev \
    mariadb-dev \
    linux-headers

# Dossier de travail
WORKDIR /app

# Copier requirements en premier pour profiter du cache
COPY requirements.txt .

# Installer les dépendances
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copier tout le reste du code
COPY . .

# Port exposé : 8080 (selon les besoins du client)
EXPOSE 8080

# Lancement de l’app avec Gunicorn en mode production
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "run:app"]
```

🔐 .dockerignore
```dockerfile
 docker
# Python
__pycache__/
*.py[cod]
*.log
*.sqlite3
*.env
*.db
*.swp
*.bak

# Environnement virtuel
venv/
env/

# IDE & outils
.vscode/
.idea/
.mypy_cache/

# Node.js (si jamais intégré pour frontend)
node_modules/

# Git
.git/
.gitignore

# Media & statiques générés
media/
static/

# Docker
Dockerfile
.dockerignore
```

# Commande de build :
```
docker build -t localhost/flask-app:latest .
```

# 📦 Déploiement des applications Kubernetes

## ⚙️ Déploiement de l'application Django (port 80)
Fichier : k8s/django-deployment.yml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: django-app
  template:
    metadata:
      labels:
        app: django-app
    spec:
      containers:
        - name: django
          image: localhost/django-app:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 5005
```

Fichier : k8s/django-service.yml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: django-service
spec:
  selector:
    app: django-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5005
  type: ClusterIP
```

## ⚙️ Déploiement de l'application Next.js (port 9090)
Fichier : k8s/next-js-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: next-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: next-app
  template:
    metadata:
      labels:
        app: next-app
    spec:
      containers:
        - name: nextjs
          image: localhost/next-js-app:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 9090
```
Fichier : k8s/next-js-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: next-service
spec:
  selector:
    app: next-app
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 9090
  type: ClusterIP
```

## ⚙️ Déploiement de l'application Flask (port 8080)

Fichier : k8s/flask-deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask
          image: localhost/flask-app:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
```

Fichier : k8s/flask-service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

## 🚀 Commandes de déploiement Kubernetes

Voici les commandes à exécuter pour déployer tous les composants dans le cluster :

```bash
# Déployer l'application Django
k apply -f k8s/django-deployment.yml
k apply -f k8s/django-service.yml

# Déployer l'application Next.js
k apply -f k8s/next-js-deployment.yml
k apply -f k8s/next-js-service.yml

# Déployer l'application Flask
k apply -f k8s/flask-deployment.yml
k apply -f k8s/flask-service.yml
```

## 🖼️ Captures d'écran du projet

### 🧩 Application 1 : Django

![image](https://github.com/user-attachments/assets/09f903af-7e0d-405e-895b-aa06b5c6c6c7)
![image](https://github.com/user-attachments/assets/c264f25f-557f-4e4c-ba0e-772e79b314c1)

### 🧩 Application 2 : Flask

![image](https://github.com/user-attachments/assets/d600b30e-71ed-454f-a298-79d11d75a8e2)
![image](https://github.com/user-attachments/assets/f248fd82-a62c-4f6d-a775-cdb279b94887)

### 🧩 Application 3 : Next.js

![image](https://github.com/user-attachments/assets/87be0d10-777e-432c-91fd-b6a593aed750)

![image](https://github.com/user-attachments/assets/154ab717-26ab-4853-a6e5-24943d8e0a0c)

## ✅ Conclusion

Ce projet nous a permis de mettre en place une maquette fonctionnelle d’un cluster Kubernetes local, capable d’héberger et de faire cohabiter trois applications web distinctes : Django (critique), Next.js, et Flask.

Grâce à l’utilisation de fichiers `Dockerfile` adaptés, d’images optimisées et de manifests Kubernetes séparés pour chaque composant, nous avons pu respecter les exigences de modularité, de portabilité et de scalabilité imposées par le client.

🎯 **Compétences mobilisées** :
- Docker & optimisation des images
- Architecture Kubernetes (Minikube, Services, etc.)
- Gestion multi-applications et exposition centralisée
- Automatisation du déploiement

Le projet est prêt pour une démonstration, un audit technique, ou une future industrialisation.
